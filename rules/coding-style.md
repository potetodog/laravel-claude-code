# Coding Style (Laravel 12.x)

## PHP Version

Laravel 12.x requires **PHP 8.2+**. Use modern PHP features:
- Constructor property promotion
- Named arguments
- Enums
- Readonly properties
- Match expressions

## PSR-12 & Laravel Pint

Use Laravel Pint for code formatting:

```bash
./vendor/bin/pint
./vendor/bin/pint --test  # Check only
```

## Strict Typing

ALWAYS use strict types in PHP files:

```php
<?php

declare(strict_types=1);

namespace App\Services;

final class UserService
{
    public function findById(int $id): ?User
    {
        return User::find($id);
    }
}
```

## DTOs with spatie/laravel-data

Use `spatie/laravel-data` for Data Transfer Objects:

```php
<?php

declare(strict_types=1);

namespace App\DTOs;

use Spatie\LaravelData\Data;

final class UserData extends Data
{
    public function __construct(
        public string $name,
        public string $email,
        public ?string $phone = null,
    ) {}
}

// Usage: Create from request
$userData = UserData::from($request);

// Usage: Create from array
$userData = UserData::from([
    'name' => 'John',
    'email' => 'john@example.com',
]);

// Usage: Transform to array
$array = $userData->toArray();
```

## File Organization

Follow Laravel conventions:
- `app/Http/Controllers/` - HTTP controllers (thin, delegate to services)
- `app/Services/` - Business logic
- `app/Models/` - Eloquent models
- `app/DTOs/` - Data Transfer Objects (using `spatie/laravel-data`)
- `app/Enums/` - PHP Enums

**Note**: This project does NOT use the Repository pattern or Actions pattern. Business logic should be placed in Services.

File size guidelines:
- Controllers: < 200 lines (prefer single action controllers)
- Services: < 400 lines
- Models: < 300 lines (extract query scopes, accessors)

## Error Handling

Use custom exceptions and Laravel's handler:

```php
// Custom exception
final class InsufficientBalanceException extends Exception
{
    public function __construct(
        public readonly int $userId,
        public readonly float $required,
        public readonly float $available,
    ) {
        parent::__construct("Insufficient balance for user {$userId}");
    }
}

// In service
public function withdraw(User $user, float $amount): void
{
    if ($user->balance < $amount) {
        throw new InsufficientBalanceException(
            $user->id,
            $amount,
            $user->balance
        );
    }
    // ...
}
```

## Input Validation

ALWAYS use Form Requests for validation:

```php
final class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    /**
     * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'string', 'min:8', 'confirmed'],
        ];
    }
}

// In controller
public function store(StoreUserRequest $request): RedirectResponse
{
    $validated = $request->validated();
    // $validated is guaranteed to be valid
}
```

## Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Controller | PascalCase, singular | `UserController` |
| Model | PascalCase, singular | `User`, `OrderItem` |
| Migration | snake_case, descriptive | `create_users_table` |
| Table | snake_case, plural | `users`, `order_items` |
| Column | snake_case | `created_at`, `user_id` |
| Method | camelCase | `getUserById()` |
| Variable | camelCase | `$orderTotal` |
| Constant | UPPER_SNAKE | `MAX_ATTEMPTS` |
| Enum | PascalCase | `OrderStatus::Pending` |

## Code Quality Checklist

Before marking work complete:
- [ ] `declare(strict_types=1)` in all PHP files
- [ ] Laravel Pint passes (`./vendor/bin/pint --test`)
- [ ] PHPStan level 4+ passes
- [ ] No `dd()`, `dump()`, or `ray()` calls
- [ ] No hardcoded values (use config/env)
- [ ] Proper type hints on all methods
- [ ] Form Requests for all input validation
- [ ] No N+1 query issues
