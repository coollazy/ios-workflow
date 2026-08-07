# ios-workflow / ios-subagents

Claude Code plugin marketplace,提供一組 iOS App 開發用的 sub agent 團隊:需求釐清 → 協調者 → 架構師 → 開發者 → 品管,涵蓋 BDD → DDD → TDD → 獨立驗收的完整循環。完整設計說明見 [`iOS App 開發 Sub Agent 團隊架構文件.md`](./iOS%20App%20開發%20Sub%20Agent%20團隊架構文件.md),流程總覽見 [`CLAUDE.md`](./CLAUDE.md)。

## 安裝

在任何專案的 Claude Code session 裡執行:

```
/plugin marketplace add https://github.com/coollazy/ios-workflow.git
/plugin install ios-subagents@ios-workflow
```

安裝後,五個 agent 會出現在該專案可用的 agent 清單裡:`requirements-agent`、`orchestrator`、`architecture-agent`、`ios-dev-agent`、`ios-qa-agent`。

## 使用方式

1. 有新功能想法或 bug 回報時,直接跟 `requirements-agent` 對話,持續問答到需求收斂,產出 BDD 規格文件
2. 規格確認「已確認待開發」後,交給 `orchestrator`,它會依序調度 `architecture-agent` → `ios-dev-agent` → `ios-qa-agent` 完成開發與驗收
3. 詳細規則(交接條件、重試上限、目錄慣例等)見各 agent 檔案本身(`plugins/ios-subagents/agents/*.md`)與 `CLAUDE.md`
