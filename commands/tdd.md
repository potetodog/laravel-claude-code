---
description: Enforce test-driven development workflow. Scaffold interfaces, generate tests FIRST, then implement minimal code to pass. Ensure 80%+ coverage.
---

# TDD Command

This command invokes the **tdd-guide** agent to enforce test-driven development methodology.

## What This Command Does

1. **Scaffold Interfaces** - Define DTOs/Contracts first
2. **Generate Tests First** - Write failing tests (RED)
3. **Implement Minimal Code** - Write just enough to pass (GREEN)
4. **Refactor** - Improve code while keeping tests green (REFACTOR)
5. **Verify Coverage** - Ensure 80%+ test coverage

## When to Use

Use `/tdd` when:
- Implementing new features
- Adding new services/actions
- Fixing bugs (write test that reproduces bug first)
- Refactoring existing code
- Building critical business logic

## How It Works

The tdd-guide agent will:

1. **Define DTOs/interfaces** for inputs/outputs
2. **Write tests that will FAIL** (because code doesn't exist yet)
3. **Run tests** and verify they fail for the right reason
4. **Write minimal implementation** to make tests pass
5. **Run tests** and verify they pass
6. **Refactor** code while keeping tests green
7. **Check coverage** and add more tests if below 80%

## TDD Cycle

```
RED → GREEN → REFACTOR → REPEAT

RED:      Write a failing test
GREEN:    Write minimal code to pass
REFACTOR: Improve code, keep tests passing
REPEAT:   Next feature/scenario
```

## Example Usage

```
User: /tdd サーベイ回答率を計算する機能を作りたい

Agent (tdd-guide):
# TDD Session: サーベイ回答率計算サービス

## Step 1: Define DTO (SCAFFOLD)

```php
<?php

declare(strict_types=1);

namespace App\DTOs;

use Spatie\LaravelData\Data;

final class SurveyStatsData extends Data
{
    public function __construct(
        public int $totalRespondents,
        public int $completedResponses,
        public float $responseRate,
    ) {}
}
```

## Step 2: Write Failing Test (RED)

```php
<?php

declare(strict_types=1);

namespace Tests\Unit\Services;

use App\DTOs\SurveyStatsData;
use App\Models\Survey;
use App\Models\SurveyResponse;
use App\Services\SurveyStatsService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class SurveyStatsServiceTest extends TestCase
{
    use RefreshDatabase;

    private SurveyStatsService $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = new SurveyStatsService();
    }

    public function test_calculates_high_response_rate(): void
    {
        $survey = Survey::factory()->create();
        SurveyResponse::factory()
            ->count(80)
            ->for($survey)
            ->completed()
            ->create();
        SurveyResponse::factory()
            ->count(20)
            ->for($survey)
            ->pending()
            ->create();

        $stats = $this->service->calculate($survey);

        $this->assertInstanceOf(SurveyStatsData::class, $stats);
        $this->assertSame(100, $stats->totalRespondents);
        $this->assertSame(80, $stats->completedResponses);
        $this->assertSame(80.0, $stats->responseRate);
    }

    public function test_calculates_low_response_rate(): void
    {
        $survey = Survey::factory()->create();
        SurveyResponse::factory()
            ->count(10)
            ->for($survey)
            ->completed()
            ->create();
        SurveyResponse::factory()
            ->count(90)
            ->for($survey)
            ->pending()
            ->create();

        $stats = $this->service->calculate($survey);

        $this->assertSame(10.0, $stats->responseRate);
    }

    public function test_handles_zero_respondents(): void
    {
        $survey = Survey::factory()->create();

        $stats = $this->service->calculate($survey);

        $this->assertSame(0, $stats->totalRespondents);
        $this->assertSame(0, $stats->completedResponses);
        $this->assertSame(0.0, $stats->responseRate);
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
php artisan test --compact --filter=SurveyStatsServiceTest

FAIL Tests\Unit\Services\SurveyStatsServiceTest
✕ test_calculates_high_response_rate
  Error: Class "App\Services\SurveyStatsService" not found

1 test failed, 0 passed
```

✅ Tests fail as expected. Ready to implement.

## Step 4: Implement Minimal Code (GREEN)

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\DTOs\SurveyStatsData;
use App\Models\Survey;

final class SurveyStatsService
{
    public function calculate(Survey $survey): SurveyStatsData
    {
        $totalRespondents = $survey->responses()->count();

        if ($totalRespondents === 0) {
            return new SurveyStatsData(
                totalRespondents: 0,
                completedResponses: 0,
                responseRate: 0.0,
            );
        }

        $completedResponses = $survey->responses()
            ->whereNotNull('completed_at')
            ->count();

        $responseRate = ($completedResponses / $totalRespondents) * 100;

        return new SurveyStatsData(
            totalRespondents: $totalRespondents,
            completedResponses: $completedResponses,
            responseRate: round($responseRate, 1),
        );
    }
}
```

## Step 5: Run Tests - Verify PASS

```bash
php artisan test --compact --filter=SurveyStatsServiceTest

PASS Tests\Unit\Services\SurveyStatsServiceTest
✓ test_calculates_high_response_rate
✓ test_calculates_low_response_rate
✓ test_handles_zero_respondents

3 tests passed
```

✅ All tests passing!

## Step 6: Refactor (IMPROVE)

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\DTOs\SurveyStatsData;
use App\Models\Survey;

final class SurveyStatsService
{
    private const DECIMAL_PRECISION = 1;

    public function calculate(Survey $survey): SurveyStatsData
    {
        $totalRespondents = $survey->responses()->count();
        $completedResponses = $this->getCompletedCount($survey);
        $responseRate = $this->calculateRate($completedResponses, $totalRespondents);

        return new SurveyStatsData(
            totalRespondents: $totalRespondents,
            completedResponses: $completedResponses,
            responseRate: $responseRate,
        );
    }

    private function getCompletedCount(Survey $survey): int
    {
        return $survey->responses()
            ->whereNotNull('completed_at')
            ->count();
    }

    private function calculateRate(int $completed, int $total): float
    {
        if ($total === 0) {
            return 0.0;
        }

        return round(($completed / $total) * 100, self::DECIMAL_PRECISION);
    }
}
```

## Step 7: Verify Tests Still Pass

```bash
php artisan test --compact --filter=SurveyStatsServiceTest

PASS Tests\Unit\Services\SurveyStatsServiceTest
✓ test_calculates_high_response_rate
✓ test_calculates_low_response_rate
✓ test_handles_zero_respondents

3 tests passed
```

✅ Refactoring complete, tests still passing!

## Step 8: Check Coverage

```bash
php artisan test --coverage --filter=SurveyStatsServiceTest

SurveyStatsService ............................ 100.0%

Coverage: 100% ✅ (Target: 80%)
```

✅ TDD session complete!
```

## TDD Best Practices

**DO:**
- ✅ Write the test FIRST, before any implementation
- ✅ Run tests and verify they FAIL before implementing
- ✅ Write minimal code to make tests pass
- ✅ Refactor only after tests are green
- ✅ Add edge cases and error scenarios
- ✅ Aim for 80%+ coverage (100% for critical code)
- ✅ Use Factories for model creation
- ✅ Use RefreshDatabase trait for database tests

**DON'T:**
- ❌ Write implementation before tests
- ❌ Skip running tests after each change
- ❌ Write too much code at once
- ❌ Ignore failing tests
- ❌ Test implementation details (test behavior)
- ❌ Create models directly without factories

## Test Types to Include

**Unit Tests** (Service/DTO-level):
- Happy path scenarios
- Edge cases (empty, null, max values)
- Error conditions
- Boundary values

**Feature Tests** (HTTP-level):
- API endpoints
- Controller actions
- Form submissions
- Authentication/authorization

**Integration Tests** (Database-level):
- Eloquent relationships
- Query scopes
- Database transactions

## Laravel Test Helpers

```php
// Authentication
$this->actingAs($user);

// HTTP Assertions
$response->assertOk();                  // 200
$response->assertCreated();             // 201
$response->assertNotFound();            // 404
$response->assertUnauthorized();        // 401
$response->assertForbidden();           // 403
$response->assertUnprocessable();       // 422

// JSON Assertions
$response->assertJson(['key' => 'value']);
$response->assertJsonStructure(['data' => ['id', 'name']]);

// Database Assertions
$this->assertDatabaseHas('users', ['email' => 'test@example.com']);
$this->assertDatabaseMissing('users', ['email' => 'deleted@example.com']);
$this->assertSoftDeleted('users', ['id' => $user->id]);

// Fakes
Mail::fake();
Queue::fake();
Notification::fake();
Storage::fake('local');
```

## Coverage Requirements

- **80% minimum** for all code
- **100% required** for:
  - Financial calculations
  - Authentication logic
  - Security-critical code
  - Core business logic

## Important Notes

**MANDATORY**: Tests must be written BEFORE implementation. The TDD cycle is:

1. **RED** - Write failing test
2. **GREEN** - Implement to pass
3. **REFACTOR** - Improve code

Never skip the RED phase. Never write code before tests.

## Integration with Other Commands

- Use `/plan` first to understand what to build
- Use `/tdd` to implement with tests
- Use `/code-review` to review implementation

## Related Agents

This command invokes the `tdd-guide` agent located at:
`~/.claude/agents/tdd-guide.md`
