# Testing Requirements (Laravel 12.x)

## Test Framework: Pest PHP

Laravel 12 uses Pest as the default testing framework:

```bash
./vendor/bin/pest              # Run all tests
./vendor/bin/pest --parallel   # Parallel execution
./vendor/bin/pest --coverage   # With coverage report
```

## Minimum Test Coverage: 80%

Test Types (ALL required):
1. **Unit Tests** - Services, Actions, DTOs, Helpers
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

## Unit Test Example

```php
<?php

declare(strict_types=1);

use App\Services\PriceCalculator;
use App\DTOs\OrderItemData;

describe('PriceCalculator', function () {
    beforeEach(function () {
        $this->calculator = new PriceCalculator();
    });

    it('calculates subtotal correctly', function () {
        $items = [
            new OrderItemData(price: 100, quantity: 2),
            new OrderItemData(price: 50, quantity: 3),
        ];

        $result = $this->calculator->subtotal($items);

        expect($result)->toBe(350);
    });

    it('applies discount percentage', function () {
        $result = $this->calculator->applyDiscount(100, 10);

        expect($result)->toBe(90);
    });

    it('throws exception for invalid discount', function () {
        expect(fn () => $this->calculator->applyDiscount(100, 150))
            ->toThrow(InvalidArgumentException::class);
    });
});
```

## Feature Test Example

```php
<?php

declare(strict_types=1);

use App\Models\User;
use App\Models\Order;

describe('Order API', function () {
    it('creates an order for authenticated user', function () {
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
    });

    it('requires authentication', function () {
        $response = $this->postJson('/api/orders', [
            'items' => [['product_id' => 1, 'quantity' => 2]],
        ]);

        $response->assertUnauthorized();
    });

    it('validates request data', function () {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/orders', []);

        $response->assertUnprocessable()
            ->assertJsonValidationErrors(['items']);
    });
});
```

## Database Testing

Use RefreshDatabase for isolation:

```php
<?php

declare(strict_types=1);

use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('soft deletes a user', function () {
    $user = User::factory()->create();

    $user->delete();

    $this->assertSoftDeleted('users', ['id' => $user->id]);
    expect(User::withTrashed()->find($user->id))->not->toBeNull();
});
```

## Mocking & Fakes

```php
<?php

use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Notification;
use App\Mail\OrderConfirmation;
use App\Jobs\ProcessOrder;

it('sends order confirmation email', function () {
    Mail::fake();

    $order = Order::factory()->create();

    // Trigger action that sends email
    app(SendOrderConfirmationAction::class)->execute($order);

    Mail::assertSent(OrderConfirmation::class, function ($mail) use ($order) {
        return $mail->hasTo($order->user->email)
            && $mail->order->id === $order->id;
    });
});

it('dispatches order processing job', function () {
    Queue::fake();

    $order = Order::factory()->create();

    app(CreateOrderAction::class)->execute($order->user, $order->items);

    Queue::assertPushed(ProcessOrder::class, fn ($job) =>
        $job->order->id === $order->id
    );
});
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
│   ├── Actions/
│   │   └── CreateOrderActionTest.php
│   ├── Services/
│   │   └── PaymentServiceTest.php
│   └── DTOs/
│       └── OrderDataTest.php
└── Pest.php
```

## Agent Support

- **tdd-guide** - Use PROACTIVELY for new features, enforces write-tests-first
- **e2e-runner** - Browser testing with Laravel Dusk
