# Laravel Claude Code

**Laravel 12.x + Inertia + React 向けにカスタマイズした Claude Code 設定集**

このリポジトリは [everything-claude-code](https://github.com/affaan-m/everything-claude-code) を Laravel 開発環境向けに修正したものです。

---

## 技術スタック

- **Backend**: Laravel 12.x (PHP 8.2+)
- **Frontend**: Inertia.js + React
- **Static Analysis**: Larastan (PHPStan)
- **Testing**: Pest PHP
- **Code Style**: Laravel Pint

---

## 構成

```
laravel-claude-code/
├── agents/           # Laravel 向けエージェント
│   ├── planner.md           # 機能実装計画
│   ├── architect.md         # システム設計
│   ├── tdd-guide.md         # Pest PHP による TDD
│   ├── code-reviewer.md     # Laravel コードレビュー
│   ├── security-reviewer.md # Laravel セキュリティ
│   ├── build-error-resolver.md  # PHPStan/Pint/Pest エラー対応
│   ├── e2e-runner.md        # Laravel Dusk + Playwright
│   ├── refactor-cleaner.md  # PHP デッドコード削除
│   └── doc-updater.md       # PHPDoc 更新
│
├── rules/            # Laravel コーディングルール
│   ├── coding-style.md     # PHP 8.2+, strict types, Pint
│   ├── patterns.md         # Action, Service, DTO パターン
│   ├── testing.md          # Pest PHP, 80% カバレッジ
│   ├── security.md         # Mass Assignment, Policy, CSRF
│   ├── performance.md      # N+1 防止, キャッシュ, Queue
│   ├── git-workflow.md     # Migration, デプロイ
│   ├── agents.md           # エージェント使用ガイド
│   └── hooks.md            # Hook 設定ガイド
│
└── hooks/            # PHP/Laravel 向け Hook
    └── hooks.json          # Pint 自動実行, dd() 警告
```

---

## 使い方

```bash
# Clone
git clone https://github.com/your-username/laravel-claude-code.git

# エージェントをコピー
cp laravel-claude-code/agents/*.md ~/.claude/agents/

# ルールをコピー
cp laravel-claude-code/rules/*.md ~/.claude/rules/

# Hooks を settings.json に追加
# hooks/hooks.json の内容を ~/.claude/settings.json にマージ
```

---

## 主な特徴

### Laravel 向けパターン
- **Action パターン**: ビジネスロジックを単一目的クラスに
- **Form Request**: 全入力のバリデーション
- **API Resource**: 一貫した JSON レスポンス
- **Policy**: 認可チェック

### 品質管理
- `./vendor/bin/pint` - コードフォーマット
- `./vendor/bin/phpstan analyse` - 静的解析
- `./vendor/bin/pest` - テスト実行

---

## クレジット

- オリジナル: [everything-claude-code](https://github.com/affaan-m/everything-claude-code) by [@affaanmustafa](https://x.com/affaanmustafa)

---

## ライセンス

MIT
