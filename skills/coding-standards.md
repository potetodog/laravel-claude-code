---
name: coding-standards
description: Universal coding standards, best practices, and patterns for PHP 8.2+, Laravel 12.x, and modern web development.
---

# Coding Standards & Best Practices

Universal coding standards applicable across Laravel projects.

## Code Quality Principles

### 1. Readability First
- Code is read more than written
- Clear variable and function names
- Self-documenting code preferred over comments
- Consistent formatting (PSR-12)

### 2. KISS (Keep It Simple, Stupid)
- Simplest solution that works
- Avoid over-engineering
- No premature optimization
- Easy to understand > clever code

### 3. DRY (Don't Repeat Yourself)
- Extract common logic into Actions/Services
- Create reusable components
- Share utilities across modules
- Avoid copy-paste programming

### 4. YAGNI (You Aren't Gonna Need It)
- Don't build features before they're needed
- Avoid speculative generality
- Add complexity only when required
- Start simple, refactor when needed

## PHP Standards

### Variable Naming

```php
// Good: Descriptive names (camelCase)
$marketSearchQuery = 'election';
$isUserAuthenticated = true;
$totalRevenue = 1000;

// Bad: Unclear names
$q = 'election';
$flag = true;
$x = 1000;
```

### Method Naming

```php
// Good: Verb-noun pattern (camelCase)
public function fetchMarketData(string $marketId): Market {}
public function calculateSimilarity(array $a, array $b): float {}
public function isValidEmail(string $email): bool {}

// Bad: Unclear or noun-only
public function market(string $id): Market {}
public function similarity($a, $b) {}
public function email($e) {}
```

### Class Naming

```php
// Good: PascalCase, descriptive
final class CreateMarketAction {}
final class MarketRepository {}
final class PaymentService {}
final readonly class CreateMarketData {}

// Bad: Unclear or wrong casing
class create_market {}
class MktRepo {}
class doPayment {}
```

### Immutability Pattern (CRITICAL)

```php
// Good: Immutable DTOs
final readonly class CreateUserData
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
    ) {}
}

// Good: Return new instances instead of mutating
public function withStatus(MarketStatus $status): self
{
    return new self(
        id: $this->id,
        name: $this->name,
        status: $status,
    );
}

// Bad: Mutable objects
class UserData
{
    public string $name;
    public function setName(string $name): void
    {
        $this->name = $name;
    }
}
```

### Error Handling

```php
// Good: Comprehensive error handling
public function fetchData(string $url): array
{
    try {
        $response = Http::timeout(30)->get($url);

        if ($response->failed()) {
            throw new HttpException("HTTP {$response->status()}: {$response->body()}");
        }

        return $response->json();
    } catch (ConnectionException $e) {
        Log::error('Connection failed', ['url' => $url, 'error' => $e->getMessage()]);
        throw new ServiceUnavailableException('External service unavailable');
    }
}

// Bad: No error handling
public function fetchData(string $url): array
{
    return Http::get($url)->json();
}
```

### Type Declarations

```php
// Good: Proper types with PHP 8.2+
final class MarketService
{
    public function __construct(
        private readonly MarketRepository $repository,
        private readonly CacheManager $cache,
    ) {}

    public function findById(int $id): ?Market
    {
        return $this->repository->find($id);
    }

    /**
     * @return Collection<int, Market>
     */
    public function getActiveMarkets(): Collection
    {
        return $this->repository->getActive();
    }
}

// Bad: Missing types
class MarketService
{
    private $repository;

    public function findById($id)
    {
        return $this->repository->find($id);
    }
}
```

### Enum Usage

```php
// Good: PHP 8.1+ Enums
enum MarketStatus: string
{
    case Active = 'active';
    case Resolved = 'resolved';
    case Closed = 'closed';

    public function label(): string
    {
        return match ($this) {
            self::Active => 'Active',
            self::Resolved => 'Resolved',
            self::Closed => 'Closed',
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::Active => 'green',
            self::Resolved => 'blue',
            self::Closed => 'gray',
        };
    }
}

// Usage
$market->status = MarketStatus::Active;
echo $market->status->label(); // "Active"
```

## Laravel Best Practices

### Controller Structure

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\Market\CreateMarketAction;
use App\DTOs\CreateMarketData;
use App\Http\Requests\StoreMarketRequest;
use App\Http\Resources\MarketResource;
use App\Models\Market;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

final class MarketController extends Controller
{
    public function __construct(
        private readonly CreateMarketAction $createAction,
    ) {}

    public function index(): AnonymousResourceCollection
    {
        $markets = Market::query()
            ->with(['user', 'categories'])
            ->latest()
            ->paginate();

        return MarketResource::collection($markets);
    }

    public function store(StoreMarketRequest $request): MarketResource
    {
        $market = $this->createAction->execute(
            CreateMarketData::fromRequest($request)
        );

        return new MarketResource($market);
    }
}
```

### Model Best Practices

```php
<?php

declare(strict_types=1);

namespace App\Models;

use App\Enums\MarketStatus;
use App\QueryBuilders\MarketQueryBuilder;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

final class Market extends Model
{
    use HasFactory;

    /**
     * @var array<int, string>
     */
    protected $fillable = [
        'user_id',
        'name',
        'description',
        'status',
        'end_date',
    ];

    /**
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'status' => MarketStatus::class,
            'end_date' => 'datetime',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function orders(): HasMany
    {
        return $this->hasMany(Order::class);
    }

    public function newEloquentBuilder($query): MarketQueryBuilder
    {
        return new MarketQueryBuilder($query);
    }
}
```

### Conditional Rendering (Blade/Inertia)

```php
// Good: Clear conditional logic
@if ($market->isActive())
    <span class="badge-active">Active</span>
@elseif ($market->isResolved())
    <span class="badge-resolved">Resolved</span>
@else
    <span class="badge-closed">Closed</span>
@endif

// Bad: Ternary hell
{{ $market->isActive() ? 'Active' : ($market->isResolved() ? 'Resolved' : 'Closed') }}
```

## API Design Standards

### REST API Conventions

```php
// routes/api.php
Route::apiResource('markets', MarketController::class);
Route::apiResource('markets.orders', MarketOrderController::class);

// Generated routes:
// GET    /api/markets              # List all markets
// GET    /api/markets/{market}     # Get specific market
// POST   /api/markets              # Create new market
// PUT    /api/markets/{market}     # Update market (full)
// PATCH  /api/markets/{market}     # Update market (partial)
// DELETE /api/markets/{market}     # Delete market

// Query parameters for filtering
// GET /api/markets?status=active&per_page=10&page=1
```

### Response Format

```php
// Good: Consistent response structure via API Resource
final class MarketResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'status' => $this->status->value,
            'created_at' => $this->created_at->toIso8601String(),
            'user' => new UserResource($this->whenLoaded('user')),
            'meta' => [
                'orders_count' => $this->whenCounted('orders'),
            ],
        ];
    }
}

// Error response (handled by Laravel)
// 422 Unprocessable Entity
{
    "message": "The name field is required.",
    "errors": {
        "name": ["The name field is required."]
    }
}
```

### Input Validation

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
        return $this->user()->can('create', Market::class);
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
            'category_ids.*' => ['integer', Rule::exists('categories', 'id')],
        ];
    }
}
```

## File Organization

### Project Structure (Laravel 12.x)

```
app/
├── Actions/           # Single-purpose action classes
│   └── Market/
│       ├── CreateMarketAction.php
│       └── UpdateMarketAction.php
├── DTOs/              # Data Transfer Objects
│   └── CreateMarketData.php
├── Enums/             # PHP Enums
│   └── MarketStatus.php
├── Events/            # Event classes
├── Exceptions/        # Custom exceptions
├── Http/
│   ├── Controllers/   # Thin controllers
│   ├── Middleware/    # HTTP middleware
│   ├── Requests/      # Form Requests (validation)
│   └── Resources/     # API Resources
├── Jobs/              # Queue jobs
├── Listeners/         # Event listeners
├── Mail/              # Mailable classes
├── Models/            # Eloquent models
├── Notifications/     # Notification classes
├── Observers/         # Model observers
├── Policies/          # Authorization policies
├── Providers/         # Service providers
├── QueryBuilders/     # Custom query builders
├── Rules/             # Custom validation rules
└── Services/          # Business logic services
```

### File Naming

```
app/Actions/CreateMarketAction.php    # PascalCase
app/DTOs/CreateMarketData.php         # PascalCase
app/Http/Requests/StoreMarketRequest.php
app/Http/Resources/MarketResource.php
app/Models/Market.php
config/markets.php                    # kebab-case
database/migrations/2025_01_01_000000_create_markets_table.php
resources/js/Pages/Markets/Index.tsx  # PascalCase for React
tests/Feature/MarketTest.php          # PascalCase
tests/Unit/CreateMarketActionTest.php
```

## Comments & Documentation

### When to Comment

```php
// Good: Explain WHY, not WHAT
// Use exponential backoff to avoid overwhelming the API during outages
$delay = min(1000 * pow(2, $retryCount), 30000);

// Deliberately bypass validation for admin imports
$market->saveQuietly();

// Bad: Stating the obvious
// Increment counter by 1
$count++;

// Set name to user's name
$name = $user->name;
```

### PHPDoc for Public APIs

```php
/**
 * Search markets using semantic similarity.
 *
 * @param string $query Natural language search query
 * @param int $limit Maximum number of results
 * @return Collection<int, Market> Markets sorted by similarity score
 *
 * @throws ServiceUnavailableException If search service is unavailable
 */
public function searchMarkets(string $query, int $limit = 10): Collection
{
    // Implementation
}
```

### Type Annotations for Static Analysis

```php
/**
 * @var Collection<int, Market>
 */
private Collection $markets;

/**
 * @param array<string, mixed> $data
 * @return array<int, Market>
 */
public function processData(array $data): array
{
    // Implementation
}
```

## Performance Best Practices

### Eager Loading

```php
// Good: Eager load relationships
$markets = Market::with(['user', 'categories', 'orders'])->get();

// Good: Conditional eager loading
$markets = Market::query()
    ->with(['user'])
    ->when($includeOrders, fn ($q) => $q->with('orders'))
    ->get();

// Bad: N+1 queries
$markets = Market::all();
foreach ($markets as $market) {
    echo $market->user->name; // N queries!
}
```

### Query Optimization

```php
// Good: Select only needed columns
$markets = Market::query()
    ->select(['id', 'name', 'status'])
    ->where('status', MarketStatus::Active)
    ->limit(10)
    ->get();

// Good: Use chunking for large datasets
Market::where('needs_processing', true)
    ->chunkById(1000, function ($markets) {
        foreach ($markets as $market) {
            ProcessMarketJob::dispatch($market);
        }
    });

// Bad: Load everything
$markets = Market::all();
```

### Caching

```php
use Illuminate\Support\Facades\Cache;

// Good: Cache expensive queries
$activeMarkets = Cache::remember(
    key: 'markets:active',
    ttl: now()->addMinutes(5),
    callback: fn () => Market::active()->with('user')->get()
);

// Good: Invalidate on changes
public function saved(Market $market): void
{
    Cache::forget('markets:active');
}
```

## Testing Standards

### Test Structure (AAA Pattern with Pest)

```php
test('creates market successfully', function () {
    // Arrange
    $user = User::factory()->create();
    $data = CreateMarketData::from([
        'name' => 'Test Market',
        'description' => 'Test Description',
        'end_date' => now()->addDays(30),
    ]);

    // Act
    $market = app(CreateMarketAction::class)->execute($data, $user);

    // Assert
    expect($market)
        ->name->toBe('Test Market')
        ->user_id->toBe($user->id);

    $this->assertDatabaseHas('markets', [
        'name' => 'Test Market',
    ]);
});
```

### Test Naming

```php
// Good: Descriptive test names
test('returns empty collection when no markets match query', function () {});
test('throws exception when API key is missing', function () {});
test('falls back to database search when Redis is unavailable', function () {});

// Bad: Vague test names
test('it works', function () {});
test('test search', function () {});
```

## Code Smell Detection

Watch for these anti-patterns:

### 1. Fat Controllers
```php
// Bad: Business logic in controllers
public function store(Request $request)
{
    // 100 lines of validation and business logic
}

// Good: Delegate to Actions/Services
public function store(StoreMarketRequest $request, CreateMarketAction $action)
{
    $market = $action->execute(CreateMarketData::fromRequest($request));
    return new MarketResource($market);
}
```

### 2. God Models
```php
// Bad: Models with 1000+ lines
class Market extends Model
{
    // Methods for everything...
}

// Good: Extract to Actions, Services, Query Builders
// Market.php stays focused on relationships and casts
```

### 3. Deep Nesting
```php
// Bad: 5+ levels of nesting
if ($user) {
    if ($user->isAdmin()) {
        if ($market) {
            if ($market->isActive()) {
                // Do something
            }
        }
    }
}

// Good: Early returns
if (!$user || !$user->isAdmin()) {
    return;
}

if (!$market || !$market->isActive()) {
    return;
}

// Do something
```

### 4. Magic Numbers
```php
// Bad: Unexplained numbers
if ($retryCount > 3) { }
sleep(500);

// Good: Named constants
private const MAX_RETRIES = 3;
private const RETRY_DELAY_MS = 500;

if ($retryCount > self::MAX_RETRIES) { }
usleep(self::RETRY_DELAY_MS * 1000);
```

### 5. Missing Type Declarations
```php
// Bad: No types
function process($data) {
    return $data;
}

// Good: Full type declarations
function process(array $data): array {
    return $data;
}
```

**Remember**: Code quality is not negotiable. Clear, maintainable code enables rapid development and confident refactoring.
