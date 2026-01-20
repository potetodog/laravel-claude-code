---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code. Enforces test-driven development with Pest PHP, including unit, feature, and integration tests with 80%+ coverage.
---

# Test-Driven Development Workflow

This skill ensures all Laravel code development follows TDD principles with comprehensive test coverage using Pest PHP.

## When to Activate

- Writing new features or functionality
- Fixing bugs or issues
- Refactoring existing code
- Adding API endpoints
- Creating new Actions/Services
- Modifying Eloquent models

## Core Principles

### 1. Tests BEFORE Code
ALWAYS write tests first, then implement code to make tests pass.

### 2. Coverage Requirements
- Minimum 80% coverage (unit + feature)
- All edge cases covered
- Error scenarios tested
- Boundary conditions verified

### 3. Test Types

#### Unit Tests
- Individual Actions and Services
- DTOs and Value Objects
- Query Builders
- Helpers and utilities

#### Feature Tests
- HTTP endpoints (API and web)
- Full request/response cycle
- Authentication flows
- Database interactions

#### Integration Tests
- External API integrations
- Queue job processing
- Event/Listener chains
- Mail and notifications

## TDD Workflow Steps

### Step 1: Write User Stories
```
As a [role], I want to [action], so that [benefit]

Example:
As a user, I want to create a market,
so that I can start accepting orders.
```

### Step 2: Generate Test Cases
For each user story, create comprehensive test cases:

```php
<?php

use App\Models\User;
use App\Models\Market;

describe('Create Market', function () {
    beforeEach(function () {
        $this->user = User::factory()->create();
    });

    test('user can create market with valid data', function () {
        // Test implementation
    });

    test('returns validation errors for invalid data', function () {
        // Test edge case
    });

    test('requires authentication', function () {
        // Test auth requirement
    });

    test('fires MarketCreated event', function () {
        // Test event dispatch
    });
});
```

### Step 3: Run Tests (They Should Fail)
```bash
php artisan test
# Tests should fail - we haven't implemented yet
```

### Step 4: Implement Code
Write minimal code to make tests pass:

```php
// Implementation guided by tests
final class CreateMarketAction
{
    public function execute(CreateMarketData $data): Market
    {
        // Implementation here
    }
}
```

### Step 5: Run Tests Again
```bash
php artisan test
# Tests should now pass
```

### Step 6: Refactor
Improve code quality while keeping tests green:
- Remove duplication
- Improve naming
- Optimize performance
- Enhance readability

### Step 7: Verify Coverage
```bash
php artisan test --coverage --min=80
# Verify 80%+ coverage achieved
```

## Testing Patterns with Pest PHP

### Unit Test Pattern
```php
<?php

use App\Actions\Market\CreateMarketAction;
use App\DTOs\CreateMarketData;
use App\Models\User;
use App\Models\Market;
use App\Models\Category;

beforeEach(function () {
    $this->user = User::factory()->create();
    $this->category = Category::factory()->create();
});

test('creates market with valid data', function () {
    // Arrange
    $data = new CreateMarketData(
        userId: $this->user->id,
        name: 'Test Market',
        description: 'Test Description',
        endDate: now()->addDays(30),
        categoryIds: [$this->category->id],
    );

    // Act
    $market = app(CreateMarketAction::class)->execute($data);

    // Assert
    expect($market)
        ->toBeInstanceOf(Market::class)
        ->name->toBe('Test Market')
        ->description->toBe('Test Description')
        ->user_id->toBe($this->user->id);

    expect($market->categories)->toHaveCount(1);

    $this->assertDatabaseHas('markets', [
        'name' => 'Test Market',
        'user_id' => $this->user->id,
    ]);
});

test('wraps operation in transaction', function () {
    $data = new CreateMarketData(
        userId: $this->user->id,
        name: 'Test Market',
        description: 'Test Description',
        endDate: now()->addDays(30),
        categoryIds: [999], // Non-existent category
    );

    expect(fn () => app(CreateMarketAction::class)->execute($data))
        ->toThrow(\Exception::class);

    $this->assertDatabaseMissing('markets', [
        'name' => 'Test Market',
    ]);
});
```

### Feature Test Pattern (HTTP)
```php
<?php

use App\Models\User;
use App\Models\Market;
use App\Models\Category;

beforeEach(function () {
    $this->user = User::factory()->create();
    $this->category = Category::factory()->create();
});

test('user can view markets list', function () {
    Market::factory()->count(3)->create();

    $response = $this->actingAs($this->user)
        ->get(route('markets.index'));

    $response
        ->assertOk()
        ->assertInertia(fn ($page) => $page
            ->component('Markets/Index')
            ->has('markets.data', 3)
        );
});

test('user can create market', function () {
    $response = $this->actingAs($this->user)
        ->post(route('markets.store'), [
            'name' => 'New Market',
            'description' => 'Market description',
            'end_date' => now()->addDays(30)->toDateString(),
            'category_ids' => [$this->category->id],
        ]);

    $response->assertRedirect();

    $this->assertDatabaseHas('markets', [
        'name' => 'New Market',
        'user_id' => $this->user->id,
    ]);
});

test('validates required fields', function () {
    $response = $this->actingAs($this->user)
        ->post(route('markets.store'), []);

    $response
        ->assertSessionHasErrors(['name', 'description', 'end_date']);
});

test('requires authentication', function () {
    $response = $this->post(route('markets.store'), [
        'name' => 'New Market',
    ]);

    $response->assertRedirect(route('login'));
});
```

### API Test Pattern
```php
<?php

use App\Models\User;
use App\Models\Market;
use Laravel\Sanctum\Sanctum;

beforeEach(function () {
    $this->user = User::factory()->create();
});

test('returns markets list', function () {
    Market::factory()->count(3)->create();

    Sanctum::actingAs($this->user);

    $response = $this->getJson(route('api.markets.index'));

    $response
        ->assertOk()
        ->assertJsonStructure([
            'data' => [
                '*' => ['id', 'name', 'description', 'status', 'created_at'],
            ],
            'links',
            'meta',
        ])
        ->assertJsonCount(3, 'data');
});

test('returns single market', function () {
    $market = Market::factory()->create(['name' => 'Test Market']);

    Sanctum::actingAs($this->user);

    $response = $this->getJson(route('api.markets.show', $market));

    $response
        ->assertOk()
        ->assertJsonPath('data.name', 'Test Market');
});

test('returns 404 for non-existent market', function () {
    Sanctum::actingAs($this->user);

    $response = $this->getJson(route('api.markets.show', 999));

    $response->assertNotFound();
});

test('requires authentication', function () {
    $response = $this->getJson(route('api.markets.index'));

    $response->assertUnauthorized();
});
```

### Authorization Test Pattern
```php
<?php

use App\Models\User;
use App\Models\Market;

test('owner can update market', function () {
    $user = User::factory()->create();
    $market = Market::factory()->create(['user_id' => $user->id]);

    $response = $this->actingAs($user)
        ->put(route('markets.update', $market), [
            'name' => 'Updated Name',
            'description' => $market->description,
            'end_date' => $market->end_date->toDateString(),
        ]);

    $response->assertRedirect();

    expect($market->fresh()->name)->toBe('Updated Name');
});

test('non-owner cannot update market', function () {
    $owner = User::factory()->create();
    $otherUser = User::factory()->create();
    $market = Market::factory()->create(['user_id' => $owner->id]);

    $response = $this->actingAs($otherUser)
        ->put(route('markets.update', $market), [
            'name' => 'Updated Name',
        ]);

    $response->assertForbidden();
});

test('admin can delete any market', function () {
    $admin = User::factory()->create(['is_admin' => true]);
    $market = Market::factory()->create();

    $response = $this->actingAs($admin)
        ->delete(route('markets.destroy', $market));

    $response->assertRedirect();

    $this->assertDatabaseMissing('markets', ['id' => $market->id]);
});
```

### Event Test Pattern
```php
<?php

use App\Events\MarketCreated;
use App\Listeners\HandleMarketCreated;
use App\Jobs\IndexMarketJob;
use App\Models\User;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Queue;

test('fires MarketCreated event when market is created', function () {
    Event::fake([MarketCreated::class]);

    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('markets.store'), [
            'name' => 'New Market',
            'description' => 'Description',
            'end_date' => now()->addDays(30)->toDateString(),
            'category_ids' => [],
        ]);

    Event::assertDispatched(MarketCreated::class, function ($event) {
        return $event->market->name === 'New Market';
    });
});

test('listener dispatches index job', function () {
    Queue::fake();

    $market = Market::factory()->create();
    $event = new MarketCreated($market);

    app(HandleMarketCreated::class)->handle($event);

    Queue::assertPushed(IndexMarketJob::class, function ($job) use ($market) {
        return $job->market->id === $market->id;
    });
});
```

### Job Test Pattern
```php
<?php

use App\Jobs\IndexMarketJob;
use App\Models\Market;
use App\Services\SearchIndexService;

test('indexes market', function () {
    $market = Market::factory()->create();

    $mockService = $this->mock(SearchIndexService::class);
    $mockService->shouldReceive('indexMarket')
        ->once()
        ->with(\Mockery::on(fn ($m) => $m->id === $market->id));

    $job = new IndexMarketJob($market);
    $job->handle($mockService);
});

test('retries on failure', function () {
    $market = Market::factory()->create();
    $job = new IndexMarketJob($market);

    expect($job->tries)->toBe(3);
    expect($job->backoff)->toBe(60);
});
```

## Test File Organization

```
tests/
├── Feature/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── MarketControllerTest.php
│   │   │   └── Api/
│   │   │       └── MarketControllerTest.php
│   │   └── Middleware/
│   ├── Actions/
│   │   └── Market/
│   │       └── CreateMarketActionTest.php
│   └── Jobs/
│       └── IndexMarketJobTest.php
├── Unit/
│   ├── DTOs/
│   │   └── CreateMarketDataTest.php
│   ├── Models/
│   │   └── MarketTest.php
│   ├── QueryBuilders/
│   │   └── MarketQueryBuilderTest.php
│   └── Services/
│       └── PaymentServiceTest.php
└── Pest.php
```

## Pest Configuration

```php
<?php
// tests/Pest.php

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

uses(TestCase::class, RefreshDatabase::class)->in('Feature');
uses(TestCase::class)->in('Unit');

expect()->extend('toBeValidMarket', function () {
    return $this
        ->toBeInstanceOf(\App\Models\Market::class)
        ->name->not->toBeEmpty()
        ->user_id->toBeInt();
});
```

## Mocking External Services

### HTTP Client Mock
```php
<?php

use Illuminate\Support\Facades\Http;

test('handles external API call', function () {
    Http::fake([
        'api.example.com/*' => Http::response([
            'data' => ['id' => 1, 'name' => 'Test'],
        ], 200),
    ]);

    $result = app(ExternalService::class)->fetchData();

    expect($result)->toBe(['id' => 1, 'name' => 'Test']);

    Http::assertSent(function ($request) {
        return $request->url() === 'https://api.example.com/data';
    });
});

test('handles API failure gracefully', function () {
    Http::fake([
        'api.example.com/*' => Http::response(null, 500),
    ]);

    expect(fn () => app(ExternalService::class)->fetchData())
        ->toThrow(ServiceUnavailableException::class);
});
```

### Cache Mock
```php
<?php

use Illuminate\Support\Facades\Cache;

test('caches result', function () {
    Cache::shouldReceive('remember')
        ->once()
        ->with('market:1', \Mockery::any(), \Mockery::any())
        ->andReturn($market = Market::factory()->make());

    $result = app(MarketRepository::class)->findById(1);

    expect($result)->toBe($market);
});
```

### Mail Mock
```php
<?php

use App\Mail\MarketCreatedMail;
use Illuminate\Support\Facades\Mail;

test('sends email when market is created', function () {
    Mail::fake();

    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('markets.store'), [
            'name' => 'New Market',
            'description' => 'Description',
            'end_date' => now()->addDays(30)->toDateString(),
            'category_ids' => [],
        ]);

    Mail::assertSent(MarketCreatedMail::class, function ($mail) use ($user) {
        return $mail->hasTo($user->email);
    });
});
```

## Test Coverage Verification

### Run Coverage Report
```bash
php artisan test --coverage
```

### Coverage Thresholds
```xml
<!-- phpunit.xml -->
<coverage processUncoveredFiles="true">
    <include>
        <directory suffix=".php">./app</directory>
    </include>
    <report>
        <html outputDirectory="coverage-report"/>
    </report>
</coverage>
```

### CI/CD Integration
```yaml
# .github/workflows/tests.yml
- name: Run Tests
  run: php artisan test --coverage --min=80

- name: Run Static Analysis
  run: ./vendor/bin/phpstan analyse
```

## Common Testing Mistakes to Avoid

### Wrong: Testing Implementation Details
```php
// Don't test internal state
expect($service->cachedData)->toBe($expectedData);
```

### Correct: Test Observable Behavior
```php
// Test what the user/system sees
expect($service->getData())->toBe($expectedData);
```

### Wrong: Brittle Database Assertions
```php
// Breaks if column order changes
$this->assertDatabaseHas('markets', $market->toArray());
```

### Correct: Specific Assertions
```php
// Only assert what matters
$this->assertDatabaseHas('markets', [
    'id' => $market->id,
    'name' => 'Expected Name',
]);
```

### Wrong: No Test Isolation
```php
// Tests depend on each other
test('creates user', function () { });
test('updates same user', function () { }); // depends on previous test
```

### Correct: Independent Tests
```php
// Each test sets up its own data
test('creates user', function () {
    $user = User::factory()->create();
    // Test logic
});

test('updates user', function () {
    $user = User::factory()->create();
    // Update logic
});
```

## Best Practices

1. **Write Tests First** - Always TDD
2. **One Assert Per Test** - Focus on single behavior
3. **Descriptive Test Names** - Explain what's tested
4. **Arrange-Act-Assert** - Clear test structure
5. **Mock External Dependencies** - Isolate unit tests
6. **Test Edge Cases** - Null, empty, boundary values
7. **Test Error Paths** - Not just happy paths
8. **Keep Tests Fast** - Unit tests < 50ms each
9. **Use Factories** - Don't hardcode test data
10. **Review Coverage Reports** - Identify gaps

## Success Metrics

- 80%+ code coverage achieved
- All tests passing (green)
- No skipped or disabled tests
- Fast test execution (< 30s for unit tests)
- Feature tests cover critical user flows
- Tests catch bugs before production

---

**Remember**: Tests are not optional. They are the safety net that enables confident refactoring, rapid development, and production reliability.
