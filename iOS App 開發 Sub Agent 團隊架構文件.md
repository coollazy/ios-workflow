# iOS App 開發 Sub Agent 團隊架構文件

## 文件目的

本文件定義一套用於「從需求釐清到開發、測試」的多 agent 協作系統，適用於 iOS App 的新功能開發與 bug fix。目標是讓每個 agent 職責邊界清楚、交接有客觀依據，最終可直接轉換為 Claude Code 的 sub agent 設定檔（`.claude/agents/*.md`）。

**範圍說明：** 本架構涵蓋「需求 → 架構 → 開發 → 測試」的循環。Release（版本管理、上架送審）目前由人工另外控制，不在本架構範圍內。

---

## 一、整體流程總覽

```
你（模糊想法 / bug 回報 / 新功能需求）
    │
    ▼
[1] Requirements Agent
    - 用 BDD 格式一直追問，直到需求收斂明確
    - 產出：Given / When / Then 格式的規格文件
    │
    ▼
[2] Orchestrator
    - 接收明確規格，拆解任務
    - 分派給 Architecture Agent → Dev Agent → QA Agent
    - 追蹤任務狀態，彙整結果回報給你
    - 不做判斷型決策（要不要做、優先級），只做執行面調度
    - 遇到需要決策的節點，停下來問你
    │
    ▼
[3] Architecture Agent
    - 依 spec 用 DDD 原則設計技術架構
    - 預設自主運作，不需要跟你來回
    - 只有「重大技術選型」（有成本/風險）才停下來問你
    - 產出：技術實作方案文件（含模組設計、資料層設計）
    │
    ▼
[4] Dev Agent
    - 依 BDD 場景，用 TDD 方式開發
      （先寫單元測試 → 寫最少 code 讓測試過 → 重構）
    - 自己的單元測試必須全過，才能標記任務完成
    - 產出：Git branch/PR，含 code + Tests/ 目錄下的單元測試
    │
    ▼  （交接：見下方「交接規則」章節）
    │
[5] QA Agent
    - 獨立依據 Requirements Agent 的 BDD 場景，重新寫驗收/整合測試
      （不讀 Dev Agent 的測試，避免球員兼裁判）
    - 跑測試、code review
    - 強制 Gate：QA 沒過，任務不算完成
    - 若失敗：產出失敗報告，退回 Dev Agent 修正（重試上限 3 次）
    - 超過重試上限：交由 Orchestrator 標記卡住，通知你介入
    │
    ▼
任務完成 → 回報給你
```

---

## 二、Agent 詳細定義

### 1. Requirements Agent

| 項目 | 內容 |
|---|---|
| **中文稱呼** | 需求釐清 |
| **系統識別名** | `requirements-agent` |
| **核心職責** | 跟你來回討論模糊需求，透過持續提問釐清細節，最終產出明確、可執行的規格 |
| **互動模式** | 高頻互動，主動引導式提問，不能自己腦補模糊處 |
| **輸出格式** | BDD（Behavior-Driven Development）格式：<br>`Given [前提條件], When [觸發動作], Then [預期結果]` |
| **產出物** | 一份或多份 BDD 場景文件，涵蓋：<br>- 功能描述<br>- 使用者情境（正常路徑 + edge case）<br>- 驗收標準 |
| **不負責** | 技術架構決策、程式碼設計 |
| **停下來問你的時機** | 每次遇到模糊或有歧義的敘述時，都要主動追問，不可自行假設 |

**追問應涵蓋的面向範例：**
- 功能邊界（這個功能包含什麼、不包含什麼）
- 異常/邊界情況（沒有網路、資料為空、使用者取消操作）
- 與現有功能的互動關係
- 是否有優先權/例外情況

---

### 2. Orchestrator

| 項目 | 內容 |
|---|---|
| **中文稱呼** | 協調者 |
| **系統識別名** | `orchestrator` |
| **核心職責** | 接收 Requirements Agent 產出的明確規格，拆解成任務並分派、追蹤進度、彙整結果 |
| **互動模式** | 不與你做需求層級的討論（那是 Requirements Agent 的工作），只在需要決策時才找你 |
| **決策權限** | **無判斷型決策權**（不決定「要不要做」「優先級高低」），純執行面調度 |
| **輸入** | Requirements Agent 產出的 BDD 規格 |
| **輸出** | 任務分派記錄、進度追蹤、彙整報告 |
| **停下來問你的時機** | 下游 agent（Architecture / Dev / QA）回報需要決策的節點時（例如重大技術選型、QA 重試超過上限）|

**未來可擴充（暫不啟用）：**
- 階段 2：可判斷任務優先級排序
- 階段 3：可判斷小範圍功能要不要做（UI 微調、非核心 bug）
- 階段 4：可自主決定大部分事情，你只看週報

> 目前版本：Orchestrator 僅做執行面調度，不具備上述決策能力。日後若要加，只需在其 system prompt 疊加決策邏輯，不需重新設計其他 agent。

---

### 3. Architecture Agent

| 項目 | 內容 |
|---|---|
| **中文稱呼** | 架構師 |
| **系統識別名** | `architecture-agent` |
| **核心職責** | 拿到 BDD 規格，判斷這個功能該怎麼接進現有 App 架構 |
| **設計原則** | DDD（Domain-Driven Design）—— 程式碼結構對應業務領域概念 |
| **互動模式** | 預設自主運作，不需要跟你來回 |
| **停下來問你的時機（僅限以下情況）** | - 需要引入新的第三方套件/框架<br>- 需要變動現有資料庫結構（可能影響已上線用戶資料）<br>- 效能 vs 開發速度的重大取捨 |
| **產出物** | 技術實作方案文件，包含：<br>- 模組/檔案結構規劃<br>- 資料層設計（如需要）<br>- 與現有架構的整合方式<br>- 驗收標準對應（供 Dev Agent 轉換為測試案例）|
| **彈性原則** | 若專案業務邏輯簡單（工具型 App），可自行判斷是否套用完整 DDD 或採輕量版，避免過度設計 |

---

### 4. Dev Agent

| 項目 | 內容 |
|---|---|
| **中文稱呼** | 開發者 |
| **系統識別名** | `ios-dev-agent` |
| **核心職責** | 依 Architecture Agent 的技術方案與 BDD 場景，撰寫 Swift/SwiftUI code |
| **開發方法** | TDD（Test-Driven Development）：<br>1. 將 BDD 場景轉換為單元測試<br>2. 寫最少的 code 讓測試通過<br>3. 重構優化 |
| **測試存放位置** | `Tests/` 目錄（單元測試，永久保留，成為回歸測試套件的一部分）|
| **交接前提** | **自己的單元測試必須全部通過**，才能標記任務為完成、才能交接給 QA Agent |
| **交接載體** | Git branch 或 PR（不可用口頭描述交接）|
| **交接單位** | 一個 BDD 場景/小任務，不累積多個場景一起交接（小步快跑原則）|
| **權限範圍** | Xcode CLI、git commit（不可直接 push to main / 不可繞過 QA Gate）|

---

### 5. QA Agent

| 項目 | 內容 |
|---|---|
| **中文稱呼** | 品管 |
| **系統識別名** | `ios-qa-agent` |
| **核心職責** | 獨立驗證 Dev Agent 的產出是否真正符合需求 |
| **測試來源** | **直接依據 Requirements Agent 的 BDD 場景**重新撰寫測試，**不讀取或複用 Dev Agent 的單元測試**（避免球員兼裁判、避免理解偏差被延續）|
| **測試層級** | Integration test / UI test（驗證完整使用者行為，而非單一函式邏輯）|
| **測試存放位置** | `IntegrationTests/`、`UITests/` 目錄（永久保留）|
| **測試命名慣例** | 建議在測試中註記對應的 BDD 場景，方便追溯，例如：<br>`// Corresponds to BDD scenario: "User logout returns to login screen"` |
| **職責** | 跑測試、code review、判斷任務是否真正完成 |
| **完成認定權** | **QA Agent 通過 = 任務完成的唯一判準**，Dev Agent 自己說完成不算數 |
| **失敗處理流程** | 1. 產出失敗報告（哪個 BDD 場景未過、期望行為 vs 實際行為、重現步驟）<br>2. 退回 Dev Agent 修正<br>3. Dev Agent 修正後重新交接<br>4. 重試上限：**3 次**<br>5. 超過上限：交由 Orchestrator 標記「任務卡住」，通知你介入（通常代表 spec 本身有問題，或 Dev/QA 對需求理解有落差）|
| **權限範圍** | xcodebuild test、唯讀 Dev Agent 產出的 code（review 用）|

---

## 三、Dev Agent ↔ QA Agent 交接規則（完整版）

| 項目 | 規則 |
|---|---|
| **交接載體** | Git branch / PR，不是口頭描述或對話紀錄 |
| **交接前提** | Dev Agent 自己的單元測試（`Tests/`）須全部通過 |
| **交接單位** | 一個 BDD 場景對應的小任務，不累積多個場景一起交接 |
| **失敗處理** | QA Agent 產出失敗報告 → 退回 Dev Agent 修正 → 重新交接 → QA 再驗一次 |
| **重試上限** | 3 次，超過交由 Orchestrator 通知你介入 |
| **完成認定** | 僅 QA Agent 通過才算完成，Dev Agent 自行宣告無效 |
| **測試獨立性** | QA 的驗收測試必須獨立於 Dev 的單元測試來源（各自依據同一份 BDD 場景獨立撰寫）|

---

## 四、開發方法論對應總表

| 方法論 | 全稱 | 解決的問題 | 對應 Agent / 環節 |
|---|---|---|---|
| **BDD** | Behavior-Driven Development | 需求要用「行為」描述清楚，確保「要做什麼」沒有模糊空間 | Requirements Agent 的產出格式 |
| **DDD** | Domain-Driven Design | 程式碼架構怎麼組織，對應業務領域概念，利於維護與擴充 | Architecture Agent 的設計原則 |
| **TDD** | Test-Driven Development | 寫 code 前先想清楚驗收條件，用測試驅動實作，確保邏輯正確 | Dev Agent 的開發手法（單元測試留存於 `Tests/`）|
| **CI（持續整合）** | Continuous Integration | 每次改動自動跑全部測試，客觀確認沒有破壞既有功能 | QA Agent 驗收 + 未來 CI 管線串接 `Tests/` + `IntegrationTests/` + `UITests/` |

> CD（持續部署）與版本發布相關流程，目前由人工另外控制，暫不納入本架構。

---

## 五、目錄結構規範

```
YourApp/
  Sources/                  ← Dev Agent 撰寫的實際程式碼
  Tests/                    ← Dev Agent 的 TDD 單元測試（永久保留，累積成回歸測試套件）
    LoginViewModelTests.swift
    UserRepositoryTests.swift
  IntegrationTests/         ← QA Agent 的整合/驗收測試（永久保留）
    LoginFlowAcceptanceTests.swift
  UITests/                  ← QA Agent 的 UI 端對端測試（永久保留）
    LoginUITests.swift
```

**規則：**
- Dev Agent 的 system prompt 須明訂：「測試永遠寫在 `Tests/`，測試單一函式/類別邏輯」
- QA Agent 的 system prompt 須明訂：「測試永遠寫在 `IntegrationTests/` 或 `UITests/`，測試完整使用者行為場景，且須對應到 Requirements Agent 的 BDD 場景」
- 所有測試檔案一律進 git 版本控制，與程式碼一起維護
- CI 每次執行時，`Tests/`、`IntegrationTests/`、`UITests/` 全部測試都要跑一次，不只跑本次改動相關的測試

---

## 六、任務類型對應的起點

| 任務類型 | 流程起點 |
|---|---|
| 新功能（New Feature）| Requirements Agent（釐清需求）→ 完整流程走一次 |
| Bug Fix | QA Agent（重現問題、定位）→ Dev Agent（用 TDD：先寫一個重現 bug 的失敗測試 → 修到通過）→ 該測試永久保留為回歸測試 |

---

## 七、尚未定案 / 未來待補充項目

| 項目 | 現況 |
|---|---|
| **Release Agent** | 已定位角色（版本管理、fastlane build、送審前需人工確認），但目前發版由人工另外控制，暫不深入設計交接規則 |
| **Orchestrator 決策權擴充** | 目前僅執行面調度，未來可視系統穩定度分階段加入優先級判斷、小範圍功能自主決策等能力 |
| **Monitoring / Bug Triage** | 建議先用排程 + 簡單腳本監控 crash rate / 評論異常，抓到異常再觸發 Orchestrator，暫不設計為常駐 LLM agent（節省成本與維護負擔）|
| **CI/CD 管線串接細節** | 尚未定義具體使用的 CI 工具（如 Xcode Cloud / GitHub Actions）與自動化觸發規則 |

---

## 八、Agent 總覽表（速查用）

| # | Agent | 系統識別名 | 中文 | 核心職責 | 互動模式 | 產出物 |
|---|---|---|---|---|---|---|
| 1 | Requirements Agent | `requirements-agent` | 需求釐清 | BDD 格式持續提問、釐清需求 | 高頻與你互動 | Given/When/Then 規格文件 |
| 2 | Orchestrator | `orchestrator` | 協調者 | 分派任務、追蹤進度、彙整回報 | 不判斷，純調度 | 任務狀態追蹤、彙整報告 |
| 3 | Architecture Agent | `architecture-agent` | 架構師 | DDD 原則設計架構 | 預設自主，重大選型才問你 | 技術實作方案文件 |
| 4 | Dev Agent | `ios-dev-agent` | 開發者 | TDD 開發（測試+code）| 自主執行 | Git branch/PR + `Tests/` |
| 5 | QA Agent | `ios-qa-agent` | 品管 | 獨立驗收測試、code review、完成認定 | 強制 Gate | 通過/失敗報告 + `IntegrationTests/`/`UITests/` |
| — | Release Agent | 待定 | 待定 | 版本管理、build、送審確認 | 待設計 | 待設計（目前人工控制）|

---

*文件版本：v1.0 — 涵蓋 Requirements → Orchestrator → Architecture → Dev → QA 五個角色的完整定義與交接規則，供轉換為 Claude Code sub agent 設定檔使用。*