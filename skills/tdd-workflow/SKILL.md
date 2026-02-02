---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code. Enforces test-driven development with PHPUnit, including unit, feature, and integration tests with 80%+ coverage.
---

# Test-Driven Development Workflow

This skill ensures all Laravel code development follows TDD principles with comprehensive test coverage using PHPUnit.

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

declare(strict_types=1);

namespace Tests\Feature\Http\Controllers;

use App\Models\User;
use App\Models\Market;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class CreateMarketTest extends TestCase
{
    use RefreshDatabase;

    private User $user;

    protected function setUp(): void
    {
        parent::setUp();
        $this->user = User::factory()->create();
    }

    public function test_user_can_create_market_with_valid_data(): void
    {
        // Test implementation
    }

    public function test_returns_validation_errors_for_invalid_data(): void
    {
        // Test edge case
    }

    public function test_requires_authentication(): void
    {
        // Test auth requirement
    }

    public function test_fires_market_created_event(): void
    {
        // Test event dispatch
    }
}
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

## Testing Patterns with PHPUnit

### Unit Test Pattern
```php
<?php

declare(strict_types=1);

namespace Tests\Unit\Actions\Market;

use App\Actions\Market\CreateMarketAction;
use App\DTOs\CreateMarketData;
use App\Models\User;
use App\Models\Market;
use App\Models\Category;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class CreateMarketActionTest extends TestCase
{
    use RefreshDatabase;

    private User $user;
    private Category $category;

    protected function setUp(): void
    {
        parent::setUp();
        $this->user = User::factory()->create();
        $this->category = Category::factory()->create();
    }

    public function test_creates_market_with_valid_data(): void
    {
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
        $this->assertInstanceOf(Market::class, $market);
        $this->assertSame('Test Market', $market->name);
        $this->assertSame('Test Description', $market->description);
        $this->assertSame($this->user->id, $market->user_id);
        $this->assertCount(1, $market->categories);

        $this->assertDatabaseHas('markets', [
            'name' => 'Test Market',
            'user_id' => $this->user->id,
        ]);
    }

    public function test_wraps_operation_in_transaction(): void
    {
        $data = new CreateMarketData(
            userId: $this->user->id,
            name: 'Test Market',
            description: 'Test Description',
            endDate: now()->addDays(30),
            categoryIds: [999], // Non-existent category
        );

        $this->expectException(\Exception::class);

        app(CreateMarketAction::class)->execute($data);

        $this->assertDatabaseMissing('markets', [
            'name' => 'Test Market',
        ]);
    }
}
```

### Feature Test Pattern (HTTP)
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Http\Controllers;

use App\Models\User;
use App\Models\Market;
use App\Models\Category;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class MarketControllerTest extends TestCase
{
    use RefreshDatabase;

    private User $user;
    private Category $category;

    protected function setUp(): void
    {
        parent::setUp();
        $this->user = User::factory()->create();
        $this->category = Category::factory()->create();
    }

    public function test_user_can_view_markets_list(): void
    {
        Market::factory()->count(3)->create();

        $response = $this->actingAs($this->user)
            ->get(route('markets.index'));

        $response
            ->assertOk()
            ->assertInertia(fn ($page) => $page
                ->component('Markets/Index')
                ->has('markets.data', 3)
            );
    }

    public function test_user_can_create_market(): void
    {
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
    }

    public function test_validates_required_fields(): void
    {
        $response = $this->actingAs($this->user)
            ->post(route('markets.store'), []);

        $response
            ->assertSessionHasErrors(['name', 'description', 'end_date']);
    }

    public function test_requires_authentication(): void
    {
        $response = $this->post(route('markets.store'), [
            'name' => 'New Market',
        ]);

        $response->assertRedirect(route('login'));
    }
}
```

### API Test Pattern
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Api;

use App\Models\User;
use App\Models\Market;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Sanctum\Sanctum;
use Tests\TestCase;

final class MarketApiTest extends TestCase
{
    use RefreshDatabase;

    private User $user;

    protected function setUp(): void
    {
        parent::setUp();
        $this->user = User::factory()->create();
    }

    public function test_returns_markets_list(): void
    {
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
    }

    public function test_returns_single_market(): void
    {
        $market = Market::factory()->create(['name' => 'Test Market']);

        Sanctum::actingAs($this->user);

        $response = $this->getJson(route('api.markets.show', $market));

        $response
            ->assertOk()
            ->assertJsonPath('data.name', 'Test Market');
    }

    public function test_returns_404_for_non_existent_market(): void
    {
        Sanctum::actingAs($this->user);

        $response = $this->getJson(route('api.markets.show', 999));

        $response->assertNotFound();
    }

    public function test_requires_authentication(): void
    {
        $response = $this->getJson(route('api.markets.index'));

        $response->assertUnauthorized();
    }
}
```

### Authorization Test Pattern
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Http\Controllers;

use App\Models\User;
use App\Models\Market;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class MarketAuthorizationTest extends TestCase
{
    use RefreshDatabase;

    public function test_owner_can_update_market(): void
    {
        $user = User::factory()->create();
        $market = Market::factory()->create(['user_id' => $user->id]);

        $response = $this->actingAs($user)
            ->put(route('markets.update', $market), [
                'name' => 'Updated Name',
                'description' => $market->description,
                'end_date' => $market->end_date->toDateString(),
            ]);

        $response->assertRedirect();

        $this->assertSame('Updated Name', $market->fresh()->name);
    }

    public function test_non_owner_cannot_update_market(): void
    {
        $owner = User::factory()->create();
        $otherUser = User::factory()->create();
        $market = Market::factory()->create(['user_id' => $owner->id]);

        $response = $this->actingAs($otherUser)
            ->put(route('markets.update', $market), [
                'name' => 'Updated Name',
            ]);

        $response->assertForbidden();
    }

    public function test_admin_can_delete_any_market(): void
    {
        $admin = User::factory()->create(['is_admin' => true]);
        $market = Market::factory()->create();

        $response = $this->actingAs($admin)
            ->delete(route('markets.destroy', $market));

        $response->assertRedirect();

        $this->assertDatabaseMissing('markets', ['id' => $market->id]);
    }
}
```

### Event Test Pattern
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Events;

use App\Events\MarketCreated;
use App\Listeners\HandleMarketCreated;
use App\Jobs\IndexMarketJob;
use App\Models\User;
use App\Models\Market;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Queue;
use Tests\TestCase;

final class MarketEventTest extends TestCase
{
    use RefreshDatabase;

    public function test_fires_market_created_event_when_market_is_created(): void
    {
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
    }

    public function test_listener_dispatches_index_job(): void
    {
        Queue::fake();

        $market = Market::factory()->create();
        $event = new MarketCreated($market);

        app(HandleMarketCreated::class)->handle($event);

        Queue::assertPushed(IndexMarketJob::class, function ($job) use ($market) {
            return $job->market->id === $market->id;
        });
    }
}
```

### Job Test Pattern
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Jobs;

use App\Jobs\IndexMarketJob;
use App\Models\Market;
use App\Services\SearchIndexService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Mockery;
use Tests\TestCase;

final class IndexMarketJobTest extends TestCase
{
    use RefreshDatabase;

    public function test_indexes_market(): void
    {
        $market = Market::factory()->create();

        $mockService = $this->mock(SearchIndexService::class);
        $mockService->shouldReceive('indexMarket')
            ->once()
            ->with(Mockery::on(fn ($m) => $m->id === $market->id));

        $job = new IndexMarketJob($market);
        $job->handle($mockService);
    }

    public function test_retries_on_failure(): void
    {
        $market = Market::factory()->create();
        $job = new IndexMarketJob($market);

        $this->assertSame(3, $job->tries);
        $this->assertSame(60, $job->backoff);
    }
}
```

## Test File Organization

```
tests/
├── Feature/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── MarketControllerTest.php
│   │   │   └── Api/
│   │   │       └── MarketApiTest.php
│   │   └── Middleware/
│   ├── Actions/
│   │   └── Market/
│   │       └── CreateMarketActionTest.php
│   ├── Events/
│   │   └── MarketEventTest.php
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
└── TestCase.php
```

## Mocking External Services

### HTTP Client Mock
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Services;

use App\Exceptions\ServiceUnavailableException;
use App\Services\ExternalService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Http;
use Tests\TestCase;

final class ExternalServiceTest extends TestCase
{
    use RefreshDatabase;

    public function test_handles_external_api_call(): void
    {
        Http::fake([
            'api.example.com/*' => Http::response([
                'data' => ['id' => 1, 'name' => 'Test'],
            ], 200),
        ]);

        $result = app(ExternalService::class)->fetchData();

        $this->assertSame(['id' => 1, 'name' => 'Test'], $result);

        Http::assertSent(function ($request) {
            return $request->url() === 'https://api.example.com/data';
        });
    }

    public function test_handles_api_failure_gracefully(): void
    {
        Http::fake([
            'api.example.com/*' => Http::response(null, 500),
        ]);

        $this->expectException(ServiceUnavailableException::class);

        app(ExternalService::class)->fetchData();
    }
}
```

### Cache Mock
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Repositories;

use App\Models\Market;
use App\Repositories\MarketRepository;
use Illuminate\Support\Facades\Cache;
use Mockery;
use Tests\TestCase;

final class MarketRepositoryTest extends TestCase
{
    public function test_caches_result(): void
    {
        $market = Market::factory()->make();

        Cache::shouldReceive('remember')
            ->once()
            ->with('market:1', Mockery::any(), Mockery::any())
            ->andReturn($market);

        $result = app(MarketRepository::class)->findById(1);

        $this->assertSame($market, $result);
    }
}
```

### Mail Mock
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Http\Controllers;

use App\Mail\MarketCreatedMail;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Mail;
use Tests\TestCase;

final class MarketNotificationTest extends TestCase
{
    use RefreshDatabase;

    public function test_sends_email_when_market_is_created(): void
    {
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
    }
}
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
$this->assertSame($expectedData, $service->cachedData);
```

### Correct: Test Observable Behavior
```php
// Test what the user/system sees
$this->assertSame($expectedData, $service->getData());
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
public function test_creates_user(): void { }
public function test_updates_same_user(): void { } // depends on previous test
```

### Correct: Independent Tests
```php
// Each test sets up its own data
public function test_creates_user(): void
{
    $user = User::factory()->create();
    // Test logic
}

public function test_updates_user(): void
{
    $user = User::factory()->create();
    // Update logic
}
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
