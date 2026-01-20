---
name: security-review
description: Use this skill when adding authentication, handling user input, working with secrets, creating API endpoints, or implementing payment/sensitive features. Provides comprehensive security checklist and patterns for Laravel.
---

# Security Review Skill

This skill ensures all Laravel code follows security best practices and identifies potential vulnerabilities.

## When to Activate

- Implementing authentication or authorization
- Handling user input or file uploads
- Creating new API endpoints
- Working with secrets or credentials
- Implementing payment features
- Storing or transmitting sensitive data
- Integrating third-party APIs

## Security Checklist

### 1. Secrets Management

#### Never Do This
```php
// Hardcoded secrets - DANGEROUS
$apiKey = "sk-proj-xxxxx";
$dbPassword = "password123";
```

#### Always Do This
```php
// Use environment variables
$apiKey = config('services.openai.key');
$dbUrl = config('database.connections.pgsql.url');

// Verify secrets exist
if (!$apiKey) {
    throw new RuntimeException('OPENAI_API_KEY not configured');
}
```

#### Verification Steps
- [ ] No hardcoded API keys, tokens, or passwords
- [ ] All secrets in `.env` file
- [ ] `.env` in `.gitignore`
- [ ] No secrets in git history
- [ ] Production secrets in hosting platform (Forge, Vapor)

### 2. Input Validation

#### Always Validate User Input with Form Requests
```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

final class StoreUserRequest extends FormRequest
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
            'email' => ['required', 'email', 'max:255', Rule::unique('users')],
            'name' => ['required', 'string', 'min:1', 'max:100'],
            'age' => ['nullable', 'integer', 'min:0', 'max:150'],
        ];
    }
}
```

#### Custom Validation Rules
```php
<?php

declare(strict_types=1);

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

final class SafeFilename implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (!preg_match('/^[\w\-. ]+$/', $value)) {
            $fail('The :attribute contains invalid characters.');
        }
    }
}
```

#### File Upload Validation
```php
public function rules(): array
{
    return [
        'avatar' => [
            'required',
            'file',
            'image',
            'mimes:jpeg,png,gif',
            'max:5120', // 5MB in kilobytes
            'dimensions:max_width=2000,max_height=2000',
        ],
    ];
}

// In controller
public function store(StoreAvatarRequest $request): RedirectResponse
{
    $path = $request->file('avatar')->store('avatars', 'public');

    $request->user()->update(['avatar_path' => $path]);

    return redirect()->back()->with('success', 'Avatar uploaded.');
}
```

#### Verification Steps
- [ ] All user inputs validated with Form Requests
- [ ] File uploads restricted (size, type, extension)
- [ ] No direct use of `$request->all()` without validation
- [ ] Whitelist validation (not blacklist)
- [ ] Error messages don't leak sensitive info

### 3. SQL Injection Prevention

#### Never Concatenate SQL
```php
// DANGEROUS - SQL Injection vulnerability
$email = $request->email;
$users = DB::select("SELECT * FROM users WHERE email = '$email'");
```

#### Always Use Eloquent or Parameterized Queries
```php
// Safe - Eloquent ORM
$user = User::where('email', $request->validated('email'))->first();

// Safe - Query Builder with bindings
$users = DB::table('users')
    ->where('email', $request->validated('email'))
    ->get();

// Safe - Raw query with bindings
$users = DB::select(
    'SELECT * FROM users WHERE email = ?',
    [$request->validated('email')]
);

// Safe - Named bindings
$users = DB::select(
    'SELECT * FROM users WHERE email = :email',
    ['email' => $request->validated('email')]
);
```

#### Verification Steps
- [ ] All database queries use Eloquent or parameterized queries
- [ ] No string concatenation in SQL
- [ ] `whereRaw()` uses bindings
- [ ] No `DB::unprepared()` with user input

### 4. Authentication & Authorization

#### Sanctum Token Handling
```php
// Good: Use Sanctum for API authentication
// config/auth.php
'guards' => [
    'api' => [
        'driver' => 'sanctum',
        'provider' => 'users',
    ],
],

// Controller
public function login(LoginRequest $request): JsonResponse
{
    $credentials = $request->validated();

    if (!Auth::attempt($credentials)) {
        throw ValidationException::withMessages([
            'email' => ['The provided credentials are incorrect.'],
        ]);
    }

    $user = Auth::user();
    $token = $user->createToken('api-token')->plainTextToken;

    return response()->json([
        'token' => $token,
        'user' => new UserResource($user),
    ]);
}
```

#### Policy-Based Authorization
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

// In Controller
public function update(UpdateMarketRequest $request, Market $market): MarketResource
{
    $this->authorize('update', $market);

    $market->update($request->validated());

    return new MarketResource($market);
}
```

#### Gate-Based Authorization
```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::define('access-admin', function (User $user): bool {
        return $user->is_admin;
    });
}

// In Controller or Middleware
if (Gate::denies('access-admin')) {
    abort(403);
}
```

#### Verification Steps
- [ ] Sanctum/Passport for API authentication
- [ ] Authorization checks via Policies/Gates
- [ ] Password hashing with bcrypt (Laravel default)
- [ ] Session management secure
- [ ] Remember token regenerated on login

### 5. XSS Prevention

#### Blade Auto-Escaping
```php
// Good: Blade auto-escapes output
<h1>{{ $user->name }}</h1>

// Dangerous: Unescaped output
<h1>{!! $user->bio !!}</h1>  // Only use for trusted HTML

// If you must render HTML, sanitize it
use Stevebauman\Purify\Facades\Purify;

<div>{!! Purify::clean($user->bio) !!}</div>
```

#### Content Security Policy
```php
// app/Http/Middleware/SecurityHeaders.php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

final class SecurityHeaders
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        $response->headers->set(
            'Content-Security-Policy',
            "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;"
        );

        $response->headers->set('X-Frame-Options', 'SAMEORIGIN');
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');

        return $response;
    }
}
```

#### Verification Steps
- [ ] Use `{{ }}` for output (auto-escaping)
- [ ] `{!! !!}` only for trusted, sanitized HTML
- [ ] CSP headers configured
- [ ] No inline JavaScript with user data

### 6. CSRF Protection

#### Laravel's Built-in CSRF Protection
```php
// Blade form
<form method="POST" action="{{ route('markets.store') }}">
    @csrf
    <!-- form fields -->
</form>

// Inertia.js handles CSRF automatically via cookies

// For AJAX requests, include the CSRF token
// resources/js/bootstrap.js
axios.defaults.headers.common['X-CSRF-TOKEN'] = document
    .querySelector('meta[name="csrf-token"]')
    ?.getAttribute('content');
```

#### Excluding Routes from CSRF (Use Sparingly)
```php
// app/Http/Middleware/VerifyCsrfToken.php
protected $except = [
    'webhooks/stripe', // Only for legitimate webhooks
];
```

#### Verification Steps
- [ ] `@csrf` directive in all forms
- [ ] CSRF token in AJAX headers
- [ ] Webhook routes verify signatures instead
- [ ] SameSite cookie attribute set

### 7. Rate Limiting

#### Define Rate Limiters
```php
// bootstrap/app.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip());
});

RateLimiter::for('search', function (Request $request) {
    return Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
});
```

#### Apply to Routes
```php
// routes/api.php
Route::middleware(['throttle:api'])->group(function () {
    Route::apiResource('markets', MarketController::class);
});

Route::middleware(['throttle:login'])->group(function () {
    Route::post('login', [AuthController::class, 'login']);
});

Route::middleware(['throttle:search'])->group(function () {
    Route::get('search', [SearchController::class, 'index']);
});
```

#### Verification Steps
- [ ] Rate limiting on all API endpoints
- [ ] Stricter limits on auth endpoints
- [ ] Stricter limits on expensive operations
- [ ] IP-based and user-based rate limiting

### 8. Sensitive Data Exposure

#### Logging
```php
// Bad: Logging sensitive data
Log::info('User login', ['email' => $email, 'password' => $password]);
Log::info('Payment', ['card_number' => $cardNumber, 'cvv' => $cvv]);

// Good: Redact sensitive data
Log::info('User login', ['email' => $email, 'user_id' => $user->id]);
Log::info('Payment', ['last4' => $card->last4, 'user_id' => $user->id]);
```

#### Error Messages
```php
// Bad: Exposing internal details
catch (\Exception $e) {
    return response()->json([
        'error' => $e->getMessage(),
        'trace' => $e->getTraceAsString(),
    ], 500);
}

// Good: Generic error messages
catch (\Exception $e) {
    Log::error('Internal error', [
        'error' => $e->getMessage(),
        'trace' => $e->getTraceAsString(),
    ]);

    return response()->json([
        'message' => 'An error occurred. Please try again.',
    ], 500);
}
```

#### Hide Sensitive Fields from JSON
```php
// app/Models/User.php
protected $hidden = [
    'password',
    'remember_token',
    'two_factor_secret',
];
```

#### Verification Steps
- [ ] No passwords, tokens, or secrets in logs
- [ ] Error messages generic for users
- [ ] Detailed errors only in logs
- [ ] Sensitive model attributes in `$hidden`
- [ ] API Resources exclude sensitive data

### 9. Mass Assignment Protection

#### Fillable vs Guarded
```php
// Good: Explicit fillable fields
final class User extends Model
{
    protected $fillable = [
        'name',
        'email',
        'password',
    ];
}

// Alternative: Guard specific fields
final class User extends Model
{
    protected $guarded = [
        'id',
        'is_admin',
        'email_verified_at',
    ];
}
```

#### Never Use Unguarded
```php
// Dangerous: Disabling mass assignment protection
Model::unguard();
User::create($request->all());  // User could set is_admin = true
Model::reguard();
```

#### Verification Steps
- [ ] All models have `$fillable` or `$guarded`
- [ ] Sensitive fields excluded from mass assignment
- [ ] No `Model::unguard()` with user input
- [ ] Use `$request->validated()` not `$request->all()`

### 10. Dependency Security

#### Regular Updates
```bash
# Check for vulnerabilities
composer audit

# Update dependencies
composer update

# Check for outdated packages
composer outdated
```

#### Lock Files
```bash
# Always commit lock files
git add composer.lock package-lock.json

# Use in CI/CD for reproducible builds
composer install --no-dev --optimize-autoloader
npm ci
```

#### Verification Steps
- [ ] Dependencies up to date
- [ ] No known vulnerabilities (`composer audit`)
- [ ] Lock files committed
- [ ] Dependabot/Renovate enabled

## Security Testing

### Automated Security Tests with Pest
```php
<?php

use App\Models\User;
use App\Models\Market;

test('requires authentication', function () {
    $this->getJson(route('api.markets.index'))
        ->assertUnauthorized();
});

test('requires authorization to update market', function () {
    $user = User::factory()->create();
    $market = Market::factory()->create(); // Different owner

    $this->actingAs($user)
        ->putJson(route('api.markets.update', $market), [
            'name' => 'Updated Name',
        ])
        ->assertForbidden();
});

test('validates input', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->postJson(route('api.markets.store'), [
            'name' => '', // Invalid
        ])
        ->assertUnprocessable()
        ->assertJsonValidationErrors(['name']);
});

test('enforces rate limits', function () {
    $user = User::factory()->create();

    // Make requests up to the limit
    for ($i = 0; $i < 60; $i++) {
        $this->actingAs($user)
            ->getJson(route('api.markets.index'));
    }

    // Next request should be rate limited
    $this->actingAs($user)
        ->getJson(route('api.markets.index'))
        ->assertStatus(429);
});

test('prevents SQL injection', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->getJson(route('api.markets.index', [
            'search' => "'; DROP TABLE markets; --",
        ]))
        ->assertOk(); // Should not error or drop table
});
```

## Pre-Deployment Security Checklist

Before ANY production deployment:

- [ ] **Secrets**: No hardcoded secrets, all in `.env`
- [ ] **Input Validation**: All inputs validated via Form Requests
- [ ] **SQL Injection**: All queries parameterized
- [ ] **XSS**: Output escaped, CSP headers configured
- [ ] **CSRF**: Protection enabled on all forms
- [ ] **Authentication**: Sanctum/Passport configured
- [ ] **Authorization**: Policies for all models
- [ ] **Rate Limiting**: Enabled on all endpoints
- [ ] **HTTPS**: Enforced in production (`APP_URL=https://...`)
- [ ] **Security Headers**: CSP, X-Frame-Options configured
- [ ] **Error Handling**: No sensitive data in errors
- [ ] **Logging**: No sensitive data logged
- [ ] **Dependencies**: Up to date, no vulnerabilities
- [ ] **Mass Assignment**: All models protected
- [ ] **CORS**: Properly configured
- [ ] **File Uploads**: Validated (size, type)
- [ ] **Debug Mode**: `APP_DEBUG=false` in production

## Resources

- [Laravel Security Documentation](https://laravel.com/docs/security)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Laravel Security Best Practices](https://laravel.com/docs/deployment#optimization)
- [Sanctum Documentation](https://laravel.com/docs/sanctum)

---

**Remember**: Security is not optional. One vulnerability can compromise the entire platform. When in doubt, err on the side of caution.
