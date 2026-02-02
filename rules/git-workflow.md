# Git Workflow (Laravel 12.x)

## Commit Message Format

```
<type>: <description>

<optional body>
```

Types: feat, fix, refactor, docs, test, chore, perf, ci

Note: Attribution disabled globally via ~/.claude/settings.json.

## Pull Request Workflow

When creating PRs:
1. Analyze full commit history (not just latest commit)
2. Use `git diff develop...HEAD` to see all changes
3. Draft comprehensive PR summary
4. Include test plan with TODOs
5. Push with `-u` flag if new branch

## Feature Implementation Workflow

1. **Plan First**
   - Use **planner** agent to create implementation plan
   - Identify dependencies and risks
   - Break down into phases

2. **Database First**
   - Create migrations for schema changes
   - Define models with relationships
   - Add factories and seeders

3. **TDD Approach**
   - Use **tdd-guide** agent
   - Write tests first (RED)
   - Implement to pass tests (GREEN)
   - Refactor (IMPROVE)
   - Verify 80%+ coverage

4. **Code Review**
   - Use **code-reviewer** agent immediately after writing code
   - Address CRITICAL and HIGH issues
   - Fix MEDIUM issues when possible

5. **Commit & Push**
   - Detailed commit messages
   - Follow conventional commits format

## Pre-Commit Checklist

Before committing:
```bash
# Format code
./vendor/bin/pint

# Run static analysis
./vendor/bin/phpstan analyse

# Run tests
./vendor/bin/pest

# Check for debug statements
git diff --cached | grep -E 'dd\(|dump\(|ray\(' && echo "Remove debug statements!"
```

## Branch Naming

```
feature/add-user-authentication
fix/order-total-calculation
refactor/extract-payment-service
chore/update-dependencies
```

## Migration Guidelines

```bash
# Create migration
php artisan make:migration create_orders_table

# Always include down() method
php artisan make:migration add_status_to_orders_table

# Run with seed in development
php artisan migrate --seed

# Fresh database in development
php artisan migrate:fresh --seed
```

Migration best practices:
- One schema change per migration
- Include rollback logic in `down()`
- Use `$table->foreignId()` for relationships
- Add indexes for frequently queried columns

## Deployment Workflow

```bash
# Pre-deployment
php artisan down --render="errors::503"

# Deploy
git pull origin main
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan queue:restart

# Post-deployment
php artisan up
```
