# Refactor Clean

Safely identify and remove dead code with test verification:

1. Run dead code analysis tools:
   - PHPStan/Larastan: 未使用コード・型エラー検出
   - composer-unused: 未使用Composer依存関係検出
   - Laravel Pint: コードスタイル問題検出

2. Generate comprehensive report in .reports/dead-code-analysis.md

3. Categorize findings by severity:
   - SAFE: テストファイル、未使用ヘルパー、未使用DTOs
   - CAUTION: Controller、Middleware、Service、Blade/Vue
   - DANGER: Config、routes、ServiceProvider、bootstrap

4. Propose safe deletions only

5. Before each deletion:
   - Run full test suite (`php artisan test --compact`)
   - Verify tests pass
   - Apply change
   - Re-run tests
   - Run `./vendor/bin/pint` for formatting
   - Rollback if tests fail

6. Show summary of cleaned items

Never delete code without running tests first!

## Laravel-Specific Checks

- 未使用のルート（`php artisan route:list` で確認）
- 未使用のMigration（既に実行済みか確認）
- 未使用のModel（関連テーブルの存在確認）
- 未使用のJob/Event/Listener
- 未使用のPolicy/Gate
- 未使用のFormRequest
- 未使用のResource/Collection
