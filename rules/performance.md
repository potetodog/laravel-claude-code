# Performance Optimization (Laravel 12.x)

## Model Selection Strategy

**Haiku 4.5** (90% of Sonnet capability, 3x cost savings):
- Lightweight agents with frequent invocation
- Pair programming and code generation
- Worker agents in multi-agent systems

**Sonnet 4.5** (Best coding model):
- Main development work
- Orchestrating multi-agent workflows
- Complex coding tasks

**Opus 4.5** (Deepest reasoning):
- Complex architectural decisions
- Maximum reasoning requirements
- Research and analysis tasks

## Eloquent N+1 Prevention

```php
// WRONG: N+1 problem
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name;  // Query per post!
}

// CORRECT: Eager loading
$posts = Post::with('author')->get();

// Multiple relations
$posts = Post::with(['author', 'comments', 'tags'])->get();

// Nested relations
$posts = Post::with('author.profile')->get();

// Conditional eager loading
$posts = Post::with(['comments' => function ($query) {
    $query->where('approved', true)->latest();
}])->get();
```

## Query Optimization

```php
// Select only needed columns
$users = User::select(['id', 'name', 'email'])->get();

// Use chunk for large datasets
User::chunk(1000, function ($users) {
    foreach ($users as $user) {
        // Process user
    }
});

// Lazy collection for memory efficiency
User::lazy()->each(function ($user) {
    // Process user
});

// Count without loading models
$count = User::where('active', true)->count();

// Exists check
if (User::where('email', $email)->exists()) {
    // ...
}
```

## Caching Strategies

```php
// Simple cache
$users = Cache::remember('users', 3600, function () {
    return User::all();
});

// Tagged cache (Redis/Memcached)
$posts = Cache::tags(['posts', 'users'])->remember("user.{$userId}.posts", 3600, function () {
    return Post::where('user_id', $userId)->get();
});

// Cache invalidation
Cache::tags(['posts'])->flush();

// Model-level caching
final class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    public function getCachedPostsAttribute(): Collection
    {
        return Cache::remember(
            "user.{$this->id}.posts",
            3600,
            fn () => $this->posts
        );
    }
}
```

## Database Indexing

```php
// In migration
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->index();
    $table->string('slug')->unique();
    $table->string('status')->index();
    $table->timestamps();

    // Composite index for common queries
    $table->index(['user_id', 'status', 'created_at']);
});
```

## Queue Usage

```php
// Dispatch heavy operations to queue
ProcessOrderJob::dispatch($order);

// Chained jobs
Bus::chain([
    new ProcessPayment($order),
    new UpdateInventory($order),
    new SendConfirmationEmail($order),
])->dispatch();

// Batch processing
Bus::batch([
    new ImportCsvChunk($chunk1),
    new ImportCsvChunk($chunk2),
    new ImportCsvChunk($chunk3),
])->dispatch();
```

## Response Optimization

```php
// API Resources with conditional attributes
final class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->when($request->user()->isAdmin(), $this->email),
            'posts_count' => $this->whenCounted('posts'),
            'posts' => PostResource::collection($this->whenLoaded('posts')),
        ];
    }
}

// Pagination
return UserResource::collection(
    User::with('posts')->paginate(15)
);
```

## Configuration Caching

```bash
# Production optimization
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Clear caches
php artisan optimize:clear
```

## Lazy Loading Prevention

```php
// In AppServiceProvider (development)
public function boot(): void
{
    Model::preventLazyLoading(! app()->isProduction());
    Model::preventSilentlyDiscardingAttributes(! app()->isProduction());
}
```

## Context Window Management

Avoid last 20% of context window for:
- Large-scale refactoring
- Feature implementation spanning multiple files
- Debugging complex interactions

Lower context sensitivity tasks:
- Single-file edits
- Independent utility creation
- Documentation updates
- Simple bug fixes

## Build Troubleshooting

If build fails:
1. Use **build-error-resolver** agent
2. Analyze error messages
3. Fix incrementally
4. Verify after each fix
