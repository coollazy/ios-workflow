# ios-workflow — iOS App 開發 Sub Agent 團隊

完整設計脈絡見 `iOS App 開發 Sub Agent 團隊架構文件.md`。本檔案是實作面的「全貌總覽」,五個 agent 的細部規則調整時,順手回來更新這裡,避免這裡跟 `.claude/agents/*.md` 的實際內容兜不起來。

## 五個 Sub Agent

定義於 `.claude/agents/`:`requirements-agent`、`orchestrator`、`architecture-agent`、`ios-dev-agent`、`ios-qa-agent`。
Release Agent 尚未定案,目前發版由人工另外控制,不在這套流程內。

## 輪轉流程

```
使用者(模糊想法 / bug 回報)
    │
    ▼
requirements-agent  ── 前景直接對話,持續提問到收斂
    │  產出 Docs/Specs/<slug>.md
    │  討論結束一律停下來跟使用者確認狀態:
    │    已確認待開發 / 暫緩 / 不做(有結論)
    │  不論結論為何,文件都保留,供日後相關議題互查
    ▼
使用者 對「已確認待開發」的規格,交給 orchestrator
    │
    ▼
orchestrator  ── 事件驅動,不高頻互動,以「一個 BDD 場景 = 一個任務」拆解
    │
    ├──▶ architecture-agent  ── 自主運作
    │      產出 Docs/Architecture/<slug>.md
    │      只有「新第三方套件 / 資料庫結構變動 / 效能取捨」才回報決策
    │
    ├──▶ ios-dev-agent  ── TDD 開發
    │      產出 Tests/ 單元測試 + Git branch/PR
    │      單元測試全過才能交接,交接單位是一個場景
    │
    └──▶ ios-qa-agent  ── 獨立驗收(不讀 dev 的單元測試)
           產出 IntegrationTests/ + UITests/
           通過 = 任務唯一完成判準
           失敗 → 退回 ios-dev-agent 修正 → 重新驗收
           重試上限 3 次,超過 → orchestrator 標記卡住,通知使用者介入
    │
    ▼
orchestrator 彙整結果 → 回報使用者
```

## 誰觸發誰(呼叫關係)

- 使用者 → `requirements-agent`(前景直接對話啟動)
- 使用者 → `orchestrator`(規格確認「已確認待開發」後手動交接)
- `orchestrator` → `architecture-agent` → `ios-dev-agent` → `ios-qa-agent`(用 Agent 工具依序背景呼叫)
- `ios-qa-agent` 失敗 → 報告回 `orchestrator` → `orchestrator` 轉給 `ios-dev-agent` 修正 → 重新走 `ios-qa-agent`

## 目錄慣例

```
Docs/
  Specs/          requirements-agent 產出的 BDD 規格文件
  Architecture/    architecture-agent 產出的技術方案文件
  QAReports/       ios-qa-agent 產出的失敗報告(<slug>-attempt-N.md)
Sources/           實際程式碼(ios-dev-agent)
Tests/             單元測試,TDD 產出,永久保留(ios-dev-agent)
IntegrationTests/  整合驗收測試,永久保留(ios-qa-agent)
UITests/           UI 端對端測試,永久保留(ios-qa-agent)
```

同一功能/bug 的 `<slug>` 在 `Specs/`、`Architecture/`、`QAReports/` 中保持一致,方便追溯同一件事在各階段的文件。

## 決策節點回報慣例

`architecture-agent`、`ios-dev-agent`、`ios-qa-agent` 都以**背景任務**形式被 `orchestrator` 呼叫,不會卡住等待即時互動。需要使用者決策時,一律在回報內容中用固定標記:

```
⚠️ 需要你的決策
```

由呼叫方(`orchestrator` 或使用者)接手詢問,agent 本身不阻塞等待回覆。

只有 `requirements-agent` 是前景直接對話(高頻來回);`orchestrator` 是事件驅動,只在下游回報上述標記,或 `ios-qa-agent` 重試滿 3 次時,才停下來問使用者。
