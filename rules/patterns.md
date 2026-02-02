# Common Patterns (Laravel 12.x)

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
