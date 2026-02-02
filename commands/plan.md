---
description: Restate requirements, assess risks, and create step-by-step implementation plan. WAIT for user CONFIRM before touching any code.
---

# Plan Command

This command invokes the **planner** agent to create a comprehensive implementation plan before writing any code.

## What This Command Does

1. **Restate Requirements** - Clarify what needs to be built
2. **Identify Risks** - Surface potential issues and blockers
3. **Create Step Plan** - Break down implementation into phases
4. **Wait for Confirmation** - MUST receive user approval before proceeding

## When to Use

Use `/plan` when:
- Starting a new feature
- Making significant architectural changes
- Working on complex refactoring
- Multiple files/components will be affected
- Requirements are unclear or ambiguous

## How It Works

The planner agent will:

1. **Analyze the request** and restate requirements in clear terms
2. **Break down into phases** with specific, actionable steps
3. **Identify dependencies** between components
4. **Assess risks** and potential blockers
5. **Estimate complexity** (High/Medium/Low)
6. **Present the plan** and WAIT for your explicit confirmation

## Example Usage

```
User: /plan サーベイ回答期限のリマインド通知機能を追加したい

Agent (planner):
# Implementation Plan: サーベイ回答期限リマインド通知

## Requirements Restatement
- 回答期限が近づいているサーベイについてユーザーに通知を送信
- 複数の通知チャネル対応（メール、Slack、アプリ内通知）
- 通知の確実な配信を保証
- ユーザーごとの通知設定を尊重

## Implementation Phases

### Phase 1: Database Schema
- notifications テーブル作成（id, user_id, survey_id, type, read_at, created_at）
- notification_preferences テーブル作成（チャネル設定用）
- user_id, survey_id にインデックス追加

### Phase 2: Notification System
- app/Notifications/SurveyReminderNotification.php 作成
- app/Services/NotificationService.php でビジネスロジック実装
- Laravel Queue でバックグラウンド処理
- 通知テンプレート作成（メール、Slack）

### Phase 3: Scheduler Integration
- app/Console/Commands/SendSurveyReminders.php 作成
- app/Console/Kernel.php でスケジュール登録
- 期限3日前、1日前、当日のリマインドロジック

### Phase 4: Frontend Components
- NotificationBell.vue コンポーネント作成
- NotificationList.vue モーダル追加
- Laravel Echo + Pusher/Reverb でリアルタイム更新
- 通知設定ページ追加

## Dependencies
- Redis（キュー用）
- Laravel Echo + Pusher または Reverb
- メール設定（SMTP/SES）

## Risks
- HIGH: メール配信率（SPF/DKIM設定必須）
- MEDIUM: 大量ユーザー時のパフォーマンス
- MEDIUM: 通知スパム防止ロジック
- LOW: リアルタイム接続のオーバーヘッド

## Estimated Complexity: MEDIUM

**WAITING FOR CONFIRMATION**: Proceed with this plan? (yes/no/modify)
```

## Important Notes

**CRITICAL**: The planner agent will **NOT** write any code until you explicitly confirm the plan with "yes" or "proceed" or similar affirmative response.

If you want changes, respond with:
- "modify: [your changes]"
- "different approach: [alternative]"
- "skip phase 2 and do phase 3 first"

## Integration with Other Commands

After planning:
- Use `/tdd` to implement with test-driven development
- Use `/code-review` to review completed implementation

## Related Agents

This command invokes the `planner` agent located at:
`~/.claude/agents/planner.md`
