---
name: backend-patterns
description: Backend architecture patterns, API design, database optimization, and server-side best practices for Laravel 12.x with PHP 8.2+.
---

# Backend Development Patterns

Backend architecture patterns and best practices for scalable Laravel applications.

## Layered Architecture

```
HTTP Request
    |
Controller (thin, validates input via Form Request)
    |
Action/Service (business logic)
    |
Model (Eloquent, data access)
    |
Database
```

## API Design Patterns

### RESTful API Structure

```php
// routes/api.php
Route::apiResource('markets', MarketController::class);

// Generated routes:
// GET    /api/markets              # index
// GET    /api/markets/{market}     # show
// POST   /api/markets              # store
// PUT    /api/markets/{market}     # update
// DELETE /api/markets/{market}     # destroy

// Query parameters for filtering, sorting, pagination
// GET /api/markets?status=active&sort=-volume&per_page=20
```

### Action Pattern (Recommended)

```php
<?php

declare(strict_types=1);

namespace App\Actions\Market;

use App\DTOs\CreateMarketData;
use App\Models\Market;
use Illuminate\Support\Facades\DB;

final class CreateMarketAction
{
    public function execute(CreateMarketData $data): Market
    {
        return DB::transaction(function () use ($data) {
            $market = Market::create([
                'user_id' => $data->userId,
                'name' => $data->name,
                'description' => $data->description,
                'end_date' => $data->endDate,
            ]);

            $market->categories()->attach($data->categoryIds);

            return $market->load('categories');
        });
    }
}
```

### DTO Pattern

```php
<?php

declare(strict_types=1);

namespace App\DTOs;

use App\Http\Requests\StoreMarketRequest;
use Carbon\Carbon;

final readonly class CreateMarketData
{
    public function __construct(
        public int $userId,
        public string $name,
        public string $description,
        public Carbon $endDate,
        public array $categoryIds = [],
    ) {}

    public static function fromRequest(StoreMarketRequest $request): self
    {
        return new self(
            userId: $request->user()->id,
            name: $request->validated('name'),
            description: $request->validated('description'),
            endDate: Carbon::parse($request->validated('end_date')),
            categoryIds: $request->validated('category_ids', []),
        );
    }
}
```

### Service Pattern

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\Market;
use App\Models\Order;
use App\Models\Transaction;
use Illuminate\Support\Facades\DB;

final class PaymentService
{
    public function __construct(
        private readonly StripeClient $stripe,
    ) {}

    public function charge(Order $order, string $paymentMethodId): Transaction
    {
        return DB::transaction(function () use ($order, $paymentMethodId) {
            $charge = $this->stripe->charges->create([
                'amount' => $order->total_cents,
                'currency' => 'jpy',
                'payment_method' => $paymentMethodId,
            ]);

            return Transaction::create([
                'order_id' => $order->id,
                'stripe_charge_id' => $charge->id,
                'amount' => $order->total_cents,
                'status' => 'completed',
            ]);
        });
    }
}
```

### Thin Controller Pattern

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Actions\Market\CreateMarketAction;
use App\DTOs\CreateMarketData;
use App\Http\Controllers\Controller;
use App\Http\Requests\StoreMarketRequest;
use App\Http\Resources\MarketResource;
use App\Models\Market;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

final class MarketController extends Controller
{
    public function index(): AnonymousResourceCollection
    {
        $markets = Market::query()
            ->with(['user', 'categories'])
            ->latest()
            ->paginate();

        return MarketResource::collection($markets);
    }

    public function store(
        StoreMarketRequest $request,
        CreateMarketAction $action
    ): MarketResource {
        $market = $action->execute(
            CreateMarketData::fromRequest($request)
        );

        return new MarketResource($market);
    }

    public function show(Market $market): MarketResource
    {
        return new MarketResource(
            $market->load(['user', 'categories', 'orders'])
        );
    }
}
```

## Form Request Validation

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

final class StoreMarketRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'min:1', 'max:200'],
            'description' => ['required', 'string', 'min:1', 'max:2000'],
            'end_date' => ['required', 'date', 'after:today'],
            'category_ids' => ['required', 'array', 'min:1'],
            'category_ids.*' => ['required', 'integer', Rule::exists('categories', 'id')],
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'name.required' => 'Market name is required.',
            'end_date.after' => 'End date must be in the future.',
        ];
    }
}
```

## API Resources

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

/**
 * @mixin \App\Models\Market
 */
final class MarketResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'status' => $this->status,
            'end_date' => $this->end_date->toIso8601String(),
            'created_at' => $this->created_at->toIso8601String(),
            'user' => new UserResource($this->whenLoaded('user')),
            'categories' => CategoryResource::collection($this->whenLoaded('categories')),
        ];
    }
}
```

## Database Patterns

### Query Optimization

```php
// Select only needed columns
$markets = Market::query()
    ->select(['id', 'name', 'status', 'volume'])
    ->where('status', 'active')
    ->orderByDesc('volume')
    ->limit(10)
    ->get();

// Bad: Select everything
$markets = Market::all();
```

### N+1 Query Prevention

```php
// Bad: N+1 query problem
$markets = Market::all();
foreach ($markets as $market) {
    echo $market->user->name; // N queries
}

// Good: Eager loading
$markets = Market::with('user')->get();
foreach ($markets as $market) {
    echo $market->user->name; // Already loaded
}

// Good: Conditional eager loading
$markets = Market::query()
    ->with(['user', 'categories'])
    ->withCount('orders')
    ->when($request->has('include_stats'), function ($query) {
        $query->withSum('orders', 'amount');
    })
    ->get();
```

### Custom Query Builder

```php
<?php

declare(strict_types=1);

namespace App\QueryBuilders;

use App\Enums\MarketStatus;
use Illuminate\Database\Eloquent\Builder;

/**
 * @extends Builder<\App\Models\Market>
 */
final class MarketQueryBuilder extends Builder
{
    public function active(): self
    {
        return $this->where('status', MarketStatus::Active);
    }

    public function forUser(int $userId): self
    {
        return $this->where('user_id', $userId);
    }

    public function endingSoon(int $days = 7): self
    {
        return $this->whereBetween('end_date', [
            now(),
            now()->addDays($days),
        ]);
    }

    public function withHighVolume(int $minVolume = 1000): self
    {
        return $this->where('volume', '>=', $minVolume);
    }
}

// Usage in Model
final class Market extends Model
{
    public function newEloquentBuilder($query): MarketQueryBuilder
    {
        return new MarketQueryBuilder($query);
    }
}

// Usage
$markets = Market::query()
    ->active()
    ->endingSoon()
    ->withHighVolume()
    ->get();
```

### Transaction Pattern

```php
use Illuminate\Support\Facades\DB;

public function createMarketWithPosition(
    CreateMarketData $marketData,
    CreatePositionData $positionData
): Market {
    return DB::transaction(function () use ($marketData, $positionData) {
        $market = Market::create($marketData->toArray());

        $market->positions()->create([
            'user_id' => $positionData->userId,
            'amount' => $positionData->amount,
            'direction' => $positionData->direction,
        ]);

        return $market->load('positions');
    });
}
```

## Caching Strategies

### Redis Caching

```php
use Illuminate\Support\Facades\Cache;

final class MarketRepository
{
    public function findById(int $id): ?Market
    {
        return Cache::remember(
            key: "market:{$id}",
            ttl: now()->addMinutes(5),
            callback: fn () => Market::find($id)
        );
    }

    public function getActiveMarkets(): Collection
    {
        return Cache::tags(['markets'])->remember(
            key: 'markets:active',
            ttl: now()->addMinutes(10),
            callback: fn () => Market::active()->get()
        );
    }

    public function invalidateCache(int $id): void
    {
        Cache::forget("market:{$id}");
        Cache::tags(['markets'])->flush();
    }
}
```

### Model Caching with Observer

```php
<?php

declare(strict_types=1);

namespace App\Observers;

use App\Models\Market;
use Illuminate\Support\Facades\Cache;

final class MarketObserver
{
    public function saved(Market $market): void
    {
        Cache::forget("market:{$market->id}");
        Cache::tags(['markets'])->flush();
    }

    public function deleted(Market $market): void
    {
        Cache::forget("market:{$market->id}");
        Cache::tags(['markets'])->flush();
    }
}
```

## Error Handling Patterns

### Custom Exception

```php
<?php

declare(strict_types=1);

namespace App\Exceptions;

use Exception;
use Illuminate\Http\JsonResponse;

final class MarketNotFoundException extends Exception
{
    public function __construct(
        public readonly int $marketId,
        string $message = 'Market not found',
    ) {
        parent::__construct($message);
    }

    public function render(): JsonResponse
    {
        return response()->json([
            'message' => $this->message,
            'market_id' => $this->marketId,
        ], 404);
    }
}
```

### Exception Handler

```php
// bootstrap/app.php
use App\Exceptions\MarketNotFoundException;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Validation\ValidationException;

return Application::configure(basePath: dirname(__DIR__))
    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (ValidationException $e) {
            return response()->json([
                'message' => 'Validation failed',
                'errors' => $e->errors(),
            ], 422);
        });

        $exceptions->render(function (MarketNotFoundException $e) {
            return $e->render();
        });
    })
    ->create();
```

## Authentication & Authorization

### Policy-Based Authorization

```php
<?php

declare(strict_types=1);

namespace App\Policies;

use App\Models\Market;
use App\Models\User;

final class MarketPolicy
{
    public function view(User $user, Market $market): bool
    {
        return true;
    }

    public function update(User $user, Market $market): bool
    {
        return $user->id === $market->user_id;
    }

    public function delete(User $user, Market $market): bool
    {
        return $user->id === $market->user_id || $user->is_admin;
    }
}

// Usage in Controller
public function update(UpdateMarketRequest $request, Market $market): MarketResource
{
    $this->authorize('update', $market);

    $market->update($request->validated());

    return new MarketResource($market);
}
```

### Gate-Based Authorization

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::define('access-admin', function (User $user): bool {
        return $user->is_admin;
    });

    Gate::define('create-market', function (User $user): bool {
        return $user->hasVerifiedEmail() && $user->markets()->count() < 10;
    });
}

// Usage
if (Gate::allows('create-market')) {
    // Create market
}
```

## Rate Limiting

```php
// bootstrap/app.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('search', function (Request $request) {
    return Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
});

// Usage in routes
Route::middleware('throttle:search')->group(function () {
    Route::get('/search', [SearchController::class, 'index']);
});
```

## Queue Jobs

### Job Definition

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Models\Market;
use App\Services\SearchIndexService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

final class IndexMarketJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60;

    public function __construct(
        public readonly Market $market,
    ) {}

    public function handle(SearchIndexService $indexService): void
    {
        $indexService->indexMarket($this->market);
    }

    public function failed(\Throwable $exception): void
    {
        report($exception);
    }
}

// Dispatch
IndexMarketJob::dispatch($market);

// Dispatch with delay
IndexMarketJob::dispatch($market)->delay(now()->addMinutes(5));

// Dispatch on specific queue
IndexMarketJob::dispatch($market)->onQueue('indexing');
```

### Job Batching

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

$markets = Market::where('needs_reindex', true)->get();

$batch = Bus::batch(
    $markets->map(fn ($market) => new IndexMarketJob($market))
)->then(function (Batch $batch) {
    Log::info('All markets indexed successfully');
})->catch(function (Batch $batch, \Throwable $e) {
    Log::error('Batch indexing failed', ['error' => $e->getMessage()]);
})->finally(function (Batch $batch) {
    Market::whereIn('id', $batch->processedJobs())
        ->update(['needs_reindex' => false]);
})->dispatch();
```

## Logging & Monitoring

### Structured Logging

```php
use Illuminate\Support\Facades\Log;

Log::info('Market created', [
    'market_id' => $market->id,
    'user_id' => $market->user_id,
    'name' => $market->name,
]);

Log::warning('Rate limit approaching', [
    'user_id' => $user->id,
    'requests_remaining' => $remaining,
]);

Log::error('Payment failed', [
    'order_id' => $order->id,
    'error' => $exception->getMessage(),
    'trace' => $exception->getTraceAsString(),
]);
```

### Custom Log Channel

```php
// config/logging.php
'channels' => [
    'payments' => [
        'driver' => 'daily',
        'path' => storage_path('logs/payments.log'),
        'level' => 'info',
        'days' => 30,
    ],
],

// Usage
Log::channel('payments')->info('Payment processed', [
    'order_id' => $order->id,
    'amount' => $order->total,
]);
```

## Event-Driven Architecture

### Event Definition

```php
<?php

declare(strict_types=1);

namespace App\Events;

use App\Models\Market;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

final class MarketCreated
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly Market $market,
    ) {}
}
```

### Listener

```php
<?php

declare(strict_types=1);

namespace App\Listeners;

use App\Events\MarketCreated;
use App\Jobs\IndexMarketJob;
use App\Notifications\MarketCreatedNotification;

final class HandleMarketCreated
{
    public function handle(MarketCreated $event): void
    {
        IndexMarketJob::dispatch($event->market);

        $event->market->user->notify(
            new MarketCreatedNotification($event->market)
        );
    }
}
```

### Event Service Provider

```php
// app/Providers/EventServiceProvider.php
protected $listen = [
    MarketCreated::class => [
        HandleMarketCreated::class,
    ],
];
```

**Remember**: Backend patterns enable scalable, maintainable server-side applications. Follow Laravel conventions and choose patterns that fit your complexity level.
