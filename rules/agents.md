# Agent Orchestration (Laravel 12.x)

## Available Agents

Located in `~/.claude/agents/`:

| Agent | Purpose | When to Use |
|-------|---------|-------------|
| planner | Implementation planning | Complex features, refactoring |
| architect | System design | Architectural decisions, API design |
| tdd-guide | Test-driven development | New features, bug fixes |
| code-reviewer | Code review | After writing code |
| security-reviewer | Security analysis | Before commits, auth changes |
| build-error-resolver | Fix build errors | When tests/Pint/PHPStan fail |
| e2e-runner | E2E testing with Dusk | Critical user flows |
| refactor-cleaner | Dead code cleanup | Code maintenance |
| doc-updater | Documentation | Updating API docs |
| migration-planner | Database changes | Schema modifications |

## Immediate Agent Usage

No user prompt needed:
1. Complex feature requests - Use **planner** agent
2. Code just written/modified - Use **code-reviewer** agent
3. Bug fix or new feature - Use **tdd-guide** agent
4. Architectural decision - Use **architect** agent
5. Database schema changes - Use **migration-planner** agent

## Parallel Task Execution

ALWAYS use parallel Task execution for independent operations:

```markdown
# GOOD: Parallel execution
Launch 3 agents in parallel:
1. Agent 1: Security analysis of AuthController.php
2. Agent 2: Performance review of OrderService.php
3. Agent 3: PHPStan analysis of app/Models/

# BAD: Sequential when unnecessary
First agent 1, then agent 2, then agent 3
```

## Laravel-Specific Agent Workflows

### New Feature Implementation
```
1. planner → Define feature scope and steps
2. migration-planner → Design database schema
3. tdd-guide → Write tests first
4. code-reviewer → Review implementation
5. security-reviewer → Security audit
```

### API Development
```
1. architect → Design API endpoints and resources
2. tdd-guide → Write feature tests
3. code-reviewer → Review controllers/resources
4. security-reviewer → Check auth/rate limiting
```

### Bug Fix
```
1. tdd-guide → Write failing test reproducing bug
2. Fix implementation
3. code-reviewer → Review fix
```

## Multi-Perspective Analysis

For complex problems, use split role sub-agents:
- Factual reviewer - Verify implementation correctness
- Senior engineer - Architecture and patterns
- Security expert - Vulnerability assessment
- Performance analyst - Query and cache optimization
- Laravel specialist - Framework best practices

## Agent Communication Patterns

```php
// When agents need to share context
// Use structured output format:

[
    'analysis' => [
        'files_reviewed' => ['app/Models/Order.php', 'app/Services/OrderService.php'],
        'issues_found' => [
            ['severity' => 'high', 'file' => 'OrderService.php', 'line' => 45, 'message' => 'N+1 query detected'],
        ],
        'recommendations' => ['Add eager loading for order items'],
    ],
]
```
