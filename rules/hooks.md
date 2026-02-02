# Hooks System (Laravel 12.x)

## Hook Types

- **PreToolUse**: Before tool execution (validation, parameter modification)
- **PostToolUse**: After tool execution (auto-format, checks)
- **Stop**: When session ends (final verification)

## Recommended Hooks (in ~/.claude/settings.json)

### PreToolUse
- **git push review**: Opens editor for review before push
- **doc blocker**: Blocks creation of unnecessary .md/.txt files

### PostToolUse
- **Laravel Pint**: Auto-formats PHP files after edit
- **PHPStan check**: Runs static analysis after editing .php files
- **dd() warning**: Warns about dd(), dump(), ray() in edited files
- **PR creation**: Logs PR URL and GitHub Actions status

### Stop
- **dd() audit**: Checks all modified files for debug statements before session ends
- **N+1 check**: Reviews queries for potential N+1 issues

## Example Hook Configuration

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "if [[ \"$CLAUDE_FILE_PATH\" == *.php ]]; then ./vendor/bin/pint \"$CLAUDE_FILE_PATH\" 2>/dev/null; fi"
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "if [[ \"$CLAUDE_FILE_PATH\" == *.php ]]; then ./vendor/bin/pint \"$CLAUDE_FILE_PATH\" 2>/dev/null; fi"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "git diff --name-only | xargs -I {} grep -l 'dd(\\|dump(\\|ray(' {} 2>/dev/null && echo 'WARNING: Debug statements found!'"
          }
        ]
      }
    ]
  }
}
```

## Auto-Accept Permissions

Use with caution:
- Enable for trusted, well-defined plans
- Disable for exploratory work
- Never use dangerously-skip-permissions flag
- Configure `allowedTools` in `~/.claude.json` instead

## Laravel Artisan Commands

Common artisan commands to allow:

```json
{
  "allowedTools": [
    "Bash(php artisan *)",
    "Bash(./vendor/bin/pint *)",
    "Bash(./vendor/bin/phpunit *)",
    "Bash(./vendor/bin/phpstan *)",
    "Bash(composer *)"
  ]
}
```

## TodoWrite Best Practices

Use TodoWrite tool to:
- Track progress on multi-step tasks
- Verify understanding of instructions
- Enable real-time steering
- Show granular implementation steps

Todo list reveals:
- Out of order steps
- Missing items
- Extra unnecessary items
- Wrong granularity
- Misinterpreted requirements

## Migration Workflow Hook

Before running migrations, always:
1. Review migration file
2. Check for potential data loss
3. Verify rollback method exists

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash(php artisan migrate*)",
      "hooks": [
        {
          "type": "confirm",
          "message": "Review migration files before running. Continue?"
        }
      ]
    }
  ]
}
```
