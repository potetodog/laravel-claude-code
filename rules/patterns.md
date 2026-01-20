# Common Patterns (Laravel 12.x)

## Action Pattern

Single-responsibility classes for business logic:

```php
<?php

declare(strict_types=1);

namespace App\Actions;

final class CreateOrderAction
{
    public function __construct(
        private readonly InventoryService $inventory,
        private readonly NotificationService $notifications,
    ) {}

    public function execute(User $user, array $items): Order
    {
        return DB::transaction(function () use ($user, $items) {
            $order = Order::create([
                'user_id' => $user->id,
                'status' => OrderStatus::Pending,
            ]);

            foreach ($items as $item) {
                $order->items()->create($item);
            }

            $this->inventory->reserve($items);
            $this->notifications->orderCreated($order);

            return $order;
        });
    }
}

// Usage in controller
public function store(StoreOrderRequest $request, CreateOrderAction $action): RedirectResponse
{
    $order = $action->execute($request->user(), $request->validated('items'));

    return redirect()->route('orders.show', $order);
}
```

## Service Pattern

Orchestrate complex business operations:

```php
<?php

declare(strict_types=1);

namespace App\Services;

final class PaymentService
{
    public function __construct(
        private readonly PaymentGateway $gateway,
    ) {}

    public function charge(Order $order, PaymentMethod $method): Transaction
    {
        $response = $this->gateway->charge(
            amount: $order->total,
            currency: $order->currency,
            method: $method,
        );

        if ($response->failed()) {
            throw new PaymentFailedException($response->error);
        }

        return Transaction::create([
            'order_id' => $order->id,
            'amount' => $order->total,
            'reference' => $response->reference,
            'status' => TransactionStatus::Completed,
        ]);
    }
}
```

## DTO Pattern

Type-safe data transfer between layers:

```php
<?php

declare(strict_types=1);

namespace App\DTOs;

final readonly class CreateUserData
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
        public ?string $phone = null,
    ) {}

    public static function fromRequest(StoreUserRequest $request): self
    {
        return new self(
            name: $request->validated('name'),
            email: $request->validated('email'),
            password: $request->validated('password'),
            phone: $request->validated('phone'),
        );
    }

    public function toArray(): array
    {
        return [
            'name' => $this->name,
            'email' => $this->email,
            'password' => Hash::make($this->password),
            'phone' => $this->phone,
        ];
    }
}
```

## Query Builder Pattern

Reusable, composable queries:

```php
<?php

declare(strict_types=1);

namespace App\QueryBuilders;

use Illuminate\Database\Eloquent\Builder;

final class OrderQueryBuilder
{
    public function __construct(
        private Builder $query,
    ) {}

    public static function make(): self
    {
        return new self(Order::query());
    }

    public function forUser(User $user): self
    {
        $this->query->where('user_id', $user->id);
        return $this;
    }

    public function status(OrderStatus $status): self
    {
        $this->query->where('status', $status);
        return $this;
    }

    public function createdBetween(Carbon $from, Carbon $to): self
    {
        $this->query->whereBetween('created_at', [$from, $to]);
        return $this;
    }

    public function withRelations(): self
    {
        $this->query->with(['items', 'user', 'payments']);
        return $this;
    }

    public function get(): Collection
    {
        return $this->query->get();
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return $this->query->paginate($perPage);
    }
}

// Usage
$orders = OrderQueryBuilder::make()
    ->forUser($user)
    ->status(OrderStatus::Pending)
    ->createdBetween(now()->subMonth(), now())
    ->withRelations()
    ->paginate();
```

## API Resource Pattern

Consistent API responses:

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

final class UserResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at->toIso8601String(),
            'orders_count' => $this->whenCounted('orders'),
            'orders' => OrderResource::collection($this->whenLoaded('orders')),
        ];
    }
}

// Collection with meta
final class UserCollection extends ResourceCollection
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total' => $this->total(),
                'per_page' => $this->perPage(),
                'current_page' => $this->currentPage(),
            ],
        ];
    }
}
```

## Observer Pattern

React to model events:

```php
<?php

declare(strict_types=1);

namespace App\Observers;

final class OrderObserver
{
    public function __construct(
        private readonly NotificationService $notifications,
        private readonly AnalyticsService $analytics,
    ) {}

    public function created(Order $order): void
    {
        $this->notifications->orderCreated($order);
        $this->analytics->trackOrder($order);
    }

    public function updated(Order $order): void
    {
        if ($order->wasChanged('status')) {
            $this->notifications->orderStatusChanged($order);
        }
    }
}

// Register in AppServiceProvider
Order::observe(OrderObserver::class);
```

## Pipeline Pattern

Chain operations cleanly:

```php
<?php

use Illuminate\Pipeline\Pipeline;

$result = app(Pipeline::class)
    ->send($order)
    ->through([
        ValidateInventory::class,
        CalculateTaxes::class,
        ApplyDiscounts::class,
        ProcessPayment::class,
        SendConfirmation::class,
    ])
    ->thenReturn();
```
