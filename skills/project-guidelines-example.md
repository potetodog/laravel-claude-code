# Project Guidelines Skill (Example)

This is an example of a project-specific skill. Use this as a template for your own Laravel projects.

---

## When to Use

Reference this skill when working on the specific project it's designed for. Project skills contain:
- Architecture overview
- File structure
- Code patterns
- Testing requirements
- Deployment workflow

---

## Architecture Overview

**Tech Stack:**
- **Backend**: Laravel 12.x (PHP 8.2+)
- **Frontend**: Inertia.js + React + TypeScript
- **Database**: PostgreSQL / MySQL
- **Cache**: Redis
- **Queue**: Redis / Database
- **Static Analysis**: Larastan (PHPStan Level 8)
- **Testing**: Pest PHP
- **Deployment**: Laravel Forge / Docker

**Services:**
```
+---------------------------------------------------------+
|                      Frontend                            |
|  Inertia.js + React + TypeScript + TailwindCSS          |
|  Rendered by Laravel, SPA-like experience               |
+---------------------------------------------------------+
                              |
                              v
+---------------------------------------------------------+
|                       Backend                            |
|  Laravel 12.x + PHP 8.2+ + Eloquent ORM                 |
|  API Resources + Form Requests + Actions                |
+---------------------------------------------------------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
        +----------+   +----------+   +----------+
        | Postgres |   |  Redis   |   |  Queue   |
        | Database |   |  Cache   |   | Workers  |
        +----------+   +----------+   +----------+
```

---

## File Structure

```
project/
├── app/
│   ├── Actions/              # Single-purpose action classes
│   │   ├── Market/
│   │   │   ├── CreateMarketAction.php
│   │   │   └── UpdateMarketAction.php
│   │   └── User/
│   ├── DTOs/                 # Data Transfer Objects
│   │   └── CreateMarketData.php
│   ├── Enums/                # PHP Enums
│   │   └── MarketStatus.php
│   ├── Events/               # Event classes
│   ├── Exceptions/           # Custom exceptions
│   ├── Http/
│   │   ├── Controllers/      # Thin controllers
│   │   ├── Middleware/       # HTTP middleware
│   │   ├── Requests/         # Form Requests (validation)
│   │   └── Resources/        # API Resources
│   ├── Jobs/                 # Queue jobs
│   ├── Listeners/            # Event listeners
│   ├── Mail/                 # Mailable classes
│   ├── Models/               # Eloquent models
│   ├── Notifications/        # Notification classes
│   ├── Observers/            # Model observers
│   ├── Policies/             # Authorization policies
│   ├── Providers/            # Service providers
│   ├── QueryBuilders/        # Custom query builders
│   ├── Rules/                # Custom validation rules
│   └── Services/             # Business logic services
│
├── config/                   # Configuration files
├── database/
│   ├── factories/            # Model factories
│   ├── migrations/           # Database migrations
│   └── seeders/              # Database seeders
│
├── resources/
│   ├── js/
│   │   ├── Components/       # Reusable React components
│   │   │   ├── UI/          # Base UI components
│   │   │   └── Forms/       # Form components
│   │   ├── Hooks/           # Custom React hooks
│   │   ├── Layouts/         # Page layouts
│   │   ├── Pages/           # Inertia pages (match routes)
│   │   │   ├── Markets/
│   │   │   │   ├── Index.tsx
│   │   │   │   ├── Show.tsx
│   │   │   │   └── Create.tsx
│   │   │   └── Auth/
│   │   ├── Types/           # TypeScript types
│   │   └── Utils/           # Utility functions
│   └── views/               # Blade views (minimal, Inertia app shell)
│
├── routes/
│   ├── web.php              # Web routes (Inertia)
│   └── api.php              # API routes
│
├── tests/
│   ├── Feature/             # Feature tests
│   ├── Unit/                # Unit tests
│   └── Pest.php             # Pest configuration
│
└── deploy/                  # Deployment configs
```

---

## Code Patterns

### Controller Pattern (Thin Controller)

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\Market\CreateMarketAction;
use App\DTOs\CreateMarketData;
use App\Http\Requests\StoreMarketRequest;
use App\Http\Resources\MarketResource;
use App\Models\Market;
use Inertia\Inertia;
use Inertia\Response;

final class MarketController extends Controller
{
    public function index(): Response
    {
        $markets = Market::query()
            ->with(['user', 'categories'])
            ->latest()
            ->paginate();

        return Inertia::render('Markets/Index', [
            'markets' => MarketResource::collection($markets),
        ]);
    }

    public function store(
        StoreMarketRequest $request,
        CreateMarketAction $action
    ): \Illuminate\Http\RedirectResponse {
        $market = $action->execute(
            CreateMarketData::fromRequest($request)
        );

        return redirect()
            ->route('markets.show', $market)
            ->with('success', 'Market created successfully.');
    }

    public function show(Market $market): Response
    {
        return Inertia::render('Markets/Show', [
            'market' => new MarketResource(
                $market->load(['user', 'categories', 'orders'])
            ),
        ]);
    }
}
```

### Action Pattern

```php
<?php

declare(strict_types=1);

namespace App\Actions\Market;

use App\DTOs\CreateMarketData;
use App\Events\MarketCreated;
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

            event(new MarketCreated($market));

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

    /**
     * @return array<string, mixed>
     */
    public function toArray(): array
    {
        return [
            'user_id' => $this->userId,
            'name' => $this->name,
            'description' => $this->description,
            'end_date' => $this->endDate,
        ];
    }
}
```

### API Resource Pattern

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
            'status' => $this->status->value,
            'status_label' => $this->status->label(),
            'end_date' => $this->end_date->toIso8601String(),
            'created_at' => $this->created_at->toIso8601String(),
            'user' => new UserResource($this->whenLoaded('user')),
            'categories' => CategoryResource::collection($this->whenLoaded('categories')),
            'orders_count' => $this->whenCounted('orders'),
        ];
    }
}
```

### Frontend Page Component (Inertia + React)

```typescript
import { Head, Link } from '@inertiajs/react'
import { MarketResource } from '@/Types/models'
import { MarketCard } from '@/Components/MarketCard'
import { Pagination } from '@/Components/UI/Pagination'

interface Props {
  markets: {
    data: MarketResource[]
    links: object
    meta: {
      current_page: number
      last_page: number
      total: number
    }
  }
}

export default function MarketsIndex({ markets }: Props) {
  return (
    <>
      <Head title="Markets" />

      <div className="container mx-auto py-8">
        <div className="flex justify-between items-center mb-6">
          <h1 className="text-2xl font-bold">Markets</h1>
          <Link
            href={route('markets.create')}
            className="px-4 py-2 bg-blue-600 text-white rounded-md"
          >
            Create Market
          </Link>
        </div>

        <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
          {markets.data.map((market) => (
            <MarketCard key={market.id} market={market} />
          ))}
        </div>

        <Pagination links={markets.links} meta={markets.meta} />
      </div>
    </>
  )
}
```

---

## Testing Requirements

### Backend (Pest PHP)

```bash
# Run all tests
php artisan test

# Run with coverage
php artisan test --coverage --min=80

# Run specific test file
php artisan test tests/Feature/MarketTest.php

# Run with parallel execution
php artisan test --parallel
```

**Test structure:**
```php
<?php

use App\Actions\Market\CreateMarketAction;
use App\DTOs\CreateMarketData;
use App\Models\User;
use App\Models\Market;

beforeEach(function () {
    $this->user = User::factory()->create();
});

test('creates market successfully', function () {
    $data = new CreateMarketData(
        userId: $this->user->id,
        name: 'Test Market',
        description: 'Test Description',
        endDate: now()->addDays(30),
        categoryIds: [],
    );

    $market = app(CreateMarketAction::class)->execute($data);

    expect($market)
        ->name->toBe('Test Market')
        ->user_id->toBe($this->user->id);

    $this->assertDatabaseHas('markets', [
        'name' => 'Test Market',
    ]);
});

test('user can view markets list', function () {
    Market::factory()->count(3)->create();

    $this->actingAs($this->user)
        ->get(route('markets.index'))
        ->assertOk()
        ->assertInertia(fn ($page) => $page
            ->component('Markets/Index')
            ->has('markets.data', 3)
        );
});

test('user can create market', function () {
    $this->actingAs($this->user)
        ->post(route('markets.store'), [
            'name' => 'New Market',
            'description' => 'Market description',
            'end_date' => now()->addDays(30)->toDateString(),
            'category_ids' => [],
        ])
        ->assertRedirect();

    $this->assertDatabaseHas('markets', [
        'name' => 'New Market',
    ]);
});
```

### Static Analysis (Larastan)

```bash
# Run Larastan
./vendor/bin/phpstan analyse

# phpstan.neon configuration
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    paths:
        - app
    level: 8
    checkMissingIterableValueType: false
```

### Frontend (Vitest)

```bash
# Run tests
npm run test

# Run with coverage
npm run test -- --coverage
```

---

## Deployment Workflow

### Pre-Deployment Checklist

- [ ] All tests passing (`php artisan test`)
- [ ] Static analysis passing (`./vendor/bin/phpstan analyse`)
- [ ] Frontend builds (`npm run build`)
- [ ] No hardcoded secrets
- [ ] Database migrations ready
- [ ] Environment variables documented

### Deployment Commands

```bash
# Production deployment
php artisan down --retry=60
git pull origin main
composer install --no-dev --optimize-autoloader
npm ci && npm run build
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan queue:restart
php artisan up
```

### Environment Variables

```bash
# .env.example
APP_NAME=Laravel
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com

DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
CACHE_STORE=redis
```

---

## Critical Rules

1. **No emojis** in code, comments, or documentation
2. **Immutability** - Use readonly DTOs, avoid mutable state
3. **TDD** - Write tests before implementation
4. **80% coverage** minimum
5. **Many small files** - 200-400 lines typical, 800 max
6. **No `dd()` or `dump()`** in production code
7. **Proper error handling** with custom exceptions
8. **Input validation** via Form Requests
9. **Authorization** via Policies
10. **Business logic** in Actions/Services, not Controllers

---

## Related Skills

- `coding-standards.md` - General coding best practices
- `backend-patterns.md` - API and database patterns
- `frontend-patterns.md` - Inertia.js and React patterns
- `security-review/` - Security best practices
- `tdd-workflow/` - Test-driven development methodology
