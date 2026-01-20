# Security Guidelines (Laravel 12.x)

## Mandatory Security Checks

Before ANY commit:
- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] All user inputs validated via Form Requests
- [ ] SQL injection prevention (Eloquent/Query Builder)
- [ ] XSS prevention (Blade escaping, `{!! !!}` avoided)
- [ ] CSRF protection on all forms
- [ ] Authentication/authorization verified (Gates/Policies)
- [ ] Rate limiting on public endpoints
- [ ] Error messages don't leak sensitive data
- [ ] Mass assignment protection ($fillable/$guarded)

## Secret Management

```php
// NEVER: Hardcoded secrets
$apiKey = "sk-proj-xxxxx";

// ALWAYS: Environment variables
$apiKey = config('services.openai.key');

// In config/services.php
'openai' => [
    'key' => env('OPENAI_API_KEY'),
],
```

## SQL Injection Prevention

```php
// WRONG: Raw queries with user input
$users = DB::select("SELECT * FROM users WHERE name = '$name'");

// CORRECT: Eloquent
$users = User::where('name', $name)->get();

// CORRECT: Query Builder with bindings
$users = DB::select('SELECT * FROM users WHERE name = ?', [$name]);

// CORRECT: Named bindings
$users = DB::select(
    'SELECT * FROM users WHERE name = :name',
    ['name' => $name]
);
```

## XSS Prevention

```blade
{{-- CORRECT: Auto-escaped --}}
<p>{{ $user->name }}</p>

{{-- WRONG: Unescaped (avoid unless absolutely necessary) --}}
<p>{!! $user->bio !!}</p>

{{-- If raw HTML needed, sanitize first --}}
<p>{!! clean($user->bio) !!}</p>
```

## Mass Assignment Protection

```php
// ALWAYS define $fillable or $guarded
final class User extends Model
{
    // Whitelist approach (recommended)
    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    // OR blacklist approach
    protected $guarded = [
        'id',
        'is_admin',
        'role',
    ];
}

// NEVER: $guarded = []
```

## Authentication & Authorization

```php
// Gates for simple authorization
Gate::define('update-post', function (User $user, Post $post) {
    return $user->id === $post->user_id;
});

// Usage
if (Gate::allows('update-post', $post)) {
    // ...
}

// Policies for model-based authorization
final class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id
            || $user->hasRole('admin');
    }
}

// In controller
public function update(UpdatePostRequest $request, Post $post): RedirectResponse
{
    $this->authorize('update', $post);
    // ...
}
```

## Rate Limiting

```php
// In routes/api.php
Route::middleware(['throttle:api'])->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
});

// Custom rate limiter in AppServiceProvider
RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip());
});

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

## CSRF Protection

```blade
{{-- In forms --}}
<form method="POST" action="/profile">
    @csrf
    @method('PUT')
    ...
</form>
```

```php
// Exclude routes from CSRF (use sparingly)
// In app/Http/Middleware/VerifyCsrfToken.php
protected $except = [
    'webhook/stripe',
    'webhook/github',
];
```

## Password Hashing

```php
// ALWAYS use Hash facade
use Illuminate\Support\Facades\Hash;

// Hashing
$hashed = Hash::make($password);

// Verification
if (Hash::check($plainPassword, $hashedPassword)) {
    // Password matches
}

// NEVER store plain passwords
// NEVER use md5(), sha1(), or similar
```

## Encryption

```php
use Illuminate\Support\Facades\Crypt;

// Encrypt sensitive data
$encrypted = Crypt::encryptString($sensitiveData);

// Decrypt
$decrypted = Crypt::decryptString($encrypted);

// For model attributes
final class User extends Model
{
    protected function casts(): array
    {
        return [
            'ssn' => 'encrypted',
            'api_token' => 'encrypted',
        ];
    }
}
```

## Secure Headers

```php
// In middleware or AppServiceProvider
public function boot(): void
{
    // Force HTTPS in production
    if (app()->environment('production')) {
        URL::forceScheme('https');
    }
}

// Content Security Policy via spatie/laravel-csp
// X-Frame-Options, X-Content-Type-Options via web server config
```

## Security Response Protocol

If security issue found:
1. STOP immediately
2. Use **security-reviewer** agent
3. Fix CRITICAL issues before continuing
4. Rotate any exposed secrets
5. Review entire codebase for similar issues
6. Add regression tests for security fixes
