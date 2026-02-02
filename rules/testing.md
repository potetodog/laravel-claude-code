# Testing Requirements (Laravel 12.x)

## Test Framework: PHPUnit

Laravel uses PHPUnit for testing:

```bash
php artisan test                    # Run all tests
php artisan test --parallel         # Parallel execution
php artisan test --coverage         # With coverage report
php artisan test --compact          # Compact output
```

## Minimum Test Coverage: 80%

Test Types (ALL required):
1. **Unit Tests** - Services, DTOs, Helpers
2. **Feature Tests** - HTTP requests, API endpoints
3. **Integration Tests** - Database operations, queues

## Test-Driven Development

MANDATORY workflow:
1. Write test first (RED)
2. Run test - it should FAIL
3. Write minimal implementation (GREEN)
4. Run test - it should PASS
5. Refactor (IMPROVE)
6. Verify coverage (80%+)

## Creating Tests

```bash
# Feature test (default)
php artisan make:test OrderControllerTest

# Unit test
php artisan make:test Services/PaymentServiceTest --unit
```

## Unit Test Example

```php
<?php

declare(strict_types=1);

namespace Tests\Unit\Services;

use App\DTOs\OrderItemData;
use App\Services\PriceCalculator;
use InvalidArgumentException;
use PHPUnit\Framework\TestCase;

final class PriceCalculatorTest extends TestCase
{
    private PriceCalculator $calculator;

    protected function setUp(): void
    {
        parent::setUp();
        $this->calculator = new PriceCalculator();
    }

    public function test_calculates_subtotal_correctly(): void
    {
        $items = [
            new OrderItemData(price: 100, quantity: 2),
            new OrderItemData(price: 50, quantity: 3),
        ];

        $result = $this->calculator->subtotal($items);

        $this->assertSame(350, $result);
    }

    public function test_applies_discount_percentage(): void
    {
        $result = $this->calculator->applyDiscount(100, 10);

        $this->assertSame(90, $result);
    }

    public function test_throws_exception_for_invalid_discount(): void
    {
        $this->expectException(InvalidArgumentException::class);

        $this->calculator->applyDiscount(100, 150);
    }
}
```

## Feature Test Example

```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Http\Controllers;

use App\Models\Order;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class OrderControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_creates_an_order_for_authenticated_user(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/orders', [
                'items' => [
                    ['product_id' => 1, 'quantity' => 2],
                ],
            ]);

        $response->assertCreated()
            ->assertJsonStructure([
                'data' => ['id', 'total', 'status', 'items'],
            ]);

        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'status' => 'pending',
        ]);
    }

    public function test_requires_authentication(): void
    {
        $response = $this->postJson('/api/orders', [
            'items' => [['product_id' => 1, 'quantity' => 2]],
        ]);

        $response->assertUnauthorized();
    }

    public function test_validates_request_data(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/orders', []);

        $response->assertUnprocessable()
            ->assertJsonValidationErrors(['items']);
    }
}
```

## Database Testing

Use RefreshDatabase for isolation:

```php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class UserDeletionTest extends TestCase
{
    use RefreshDatabase;

    public function test_soft_deletes_a_user(): void
    {
        $user = User::factory()->create();

        $user->delete();

        $this->assertSoftDeleted('users', ['id' => $user->id]);
        $this->assertNotNull(User::withTrashed()->find($user->id));
    }
}
```

## Mocking & Fakes

```php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use App\Jobs\ProcessOrder;
use App\Mail\OrderConfirmation;
use App\Models\Order;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Queue;
use Tests\TestCase;

final class OrderNotificationTest extends TestCase
{
    use RefreshDatabase;

    public function test_sends_order_confirmation_email(): void
    {
        Mail::fake();

        $order = Order::factory()->create();

        // Trigger action that sends email
        app(SendOrderConfirmationAction::class)->execute($order);

        Mail::assertSent(OrderConfirmation::class, function ($mail) use ($order) {
            return $mail->hasTo($order->user->email)
                && $mail->order->id === $order->id;
        });
    }

    public function test_dispatches_order_processing_job(): void
    {
        Queue::fake();

        $order = Order::factory()->create();

        app(CreateOrderAction::class)->execute($order->user, $order->items);

        Queue::assertPushed(ProcessOrder::class, fn ($job) =>
            $job->order->id === $order->id
        );
    }
}
```

## HTTP Test Helpers

```php
// Authentication
$this->actingAs($user);
$this->actingAs($user, 'api');

// Assertions
$response->assertOk();                    // 200
$response->assertCreated();               // 201
$response->assertNoContent();             // 204
$response->assertNotFound();              // 404
$response->assertUnauthorized();          // 401
$response->assertForbidden();             // 403
$response->assertUnprocessable();         // 422

// JSON assertions
$response->assertJson(['key' => 'value']);
$response->assertJsonStructure(['data' => ['id', 'name']]);
$response->assertJsonCount(3, 'data');
$response->assertJsonPath('data.0.id', 1);
```

## Test Organization

```
tests/
├── Feature/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── OrderControllerTest.php
│   │   │   └── UserControllerTest.php
│   │   └── Middleware/
│   │       └── AuthenticationTest.php
│   └── Api/
│       └── V1/
│           └── OrderApiTest.php
├── Unit/
│   ├── Services/
│   │   └── PaymentServiceTest.php
│   └── DTOs/
│       └── OrderDataTest.php
└── TestCase.php
```

## Running Tests

```bash
# Run all tests
php artisan test --compact

# Run specific test file
php artisan test --compact tests/Feature/Http/Controllers/OrderControllerTest.php

# Filter by test method name
php artisan test --compact --filter=test_creates_an_order

# Run with coverage
php artisan test --coverage --min=80
```

## Agent Support

- **tdd-guide** - Use PROACTIVELY for new features, enforces write-tests-first
