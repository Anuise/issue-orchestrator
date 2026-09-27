# Issue Orchestrator

一個可重複使用的 coding-agent skill，依據規格文件與本地 Markdown issue 完成一個 feature。每個可重啟的 controller epoch 會挑選一個 ready issue，交給全新、隔離的實作 agent，再由全新的唯讀 verifier 檢查結果，持久化一次 durable 狀態轉換，然後讓位給下一個 epoch。

Controller 只編輯追蹤用的 metadata。每個 worker 只實作一個 issue，每次修正也都使用新的 worker。v1 刻意在單一 checkout 中循序執行，不使用 worktree、merge agent、GitHub Issues，也不會自動建立 PR 或 push。

## 安裝

在任一專案目錄中以互動方式安裝：

```sh
npx skills@latest add Anuise/issue-orchestrator
```

只選擇此 skill：

```sh
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator
```

明確指定 agent（預設為專案範圍）：

```sh
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent claude-code
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent codex
```

全域安裝時加上 `--global`：

```sh
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent claude-code --global
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent codex --global
```

`--yes` 可略過安裝程式的提示，`--copy` 以複製取代連結。不安裝而列出此 repository 的 skill、列出已安裝的 skill，或搜尋 skills 目錄：

```sh
npx skills@latest add Anuise/issue-orchestrator --list
npx skills@latest list
npx skills@latest find issue-orchestrator
```

更新已安裝的 skill，可選擇指定此 skill 與其範圍：

```sh
npx skills@latest update
npx skills@latest update issue-orchestrator --project --yes
npx skills@latest update issue-orchestrator --global --yes
```

指令使用以空格分隔的 `--skill issue-orchestrator` 語法，已對照 `skills` **1.7.0** 檢查。兩個 agent 識別名稱都經過實際的本地安裝驗證。全域與更新語法依 CLI help 檢查；驗證過程不會變更你的全域 agent 設定。`@latest` 可能變動；若未來版本不同，請參考 `npx skills@latest --help`。詳見[驗證紀錄](docs/validation.md)與[上游 CLI 文件](https://github.com/vercel-labs/skills)。

## 使用

準備 feature 工作區：

```text
.scratch/example-feature/
├── spec.md
└── issues/
    ├── 01-field-verify-downloads.md
    ├── 02-field-verify-writes.md
    └── 03-list-both-job-families.md
```

每個 issue 需有驗收條件（acceptance criteria），並在已知時明確列出相依關係：

```markdown
# 03 List both job families
Status: pending
Depends on: 01-field-verify-downloads.md

## Acceptance criteria
- The list includes both job families with their verified download fields.
- A behavioral test covers one record from each family.
```

在支援 slash command 呼叫 skill 的 runtime 中：

```text
/issue-orchestrator .scratch/example-feature
```

可攜的自然語言呼叫方式：

```text
Use issue-orchestrator on .scratch/example-feature.
Continue until all executable issues are complete or genuinely blocked.
```

路徑若非絕對路徑，則相對於目前 repository 根目錄解析。除非 spec 或使用者明確排除，所有 issue 檔案皆為必要。既有的 repository 指示仍然規範測試、commit 權限與實作慣例。

## 架構

對話歷史只是快取；durable 狀態才是權威來源。沒有任何 agent 會在整個 feature 期間持續存活：

```text
spec + issues + Git → fingerprints ─┬─ unchanged ──────────────────┐
                                    └─ changed → fresh scheduler ──┤
                                                                   ↓
                                                 .orchestration/state.json
                                                                   ↓
          fresh controller epoch → choose ONE ready issue → fresh worker
                   ↑                                             ↓ result.json
          persist one transition ← verification.json ← fresh read-only verifier
                   ↓ when all issues are verified
          fresh read-only integration verifier → integration/<run>/result.json
```

- **Controller epoch。** 每個 epoch 讀取 `state.json`、驗證 fingerprint、處理任何進行中的 attempt、執行一個 worker 與一個 verifier，對 `state.json` 與 issue 檔案套用一次 write-ahead 轉換，然後讓位。只要有 repository、feature 路徑、skill 與磁碟狀態，全新的 controller 就能接續執行。
- **`state.json`。** 位於 `<feature>/.orchestration/state.json`，由 controller 單獨寫入。記錄 fingerprint、branch/HEAD、相依圖與排序、各 issue 狀態與 blocker、進行中的 attempt 及其 phase、`pending_transition`、`last_attempt`，以及 integration 狀態。先前的選擇、worker 說明、修正歷史與完成證據都從磁碟與 Git 取得，不依賴記憶。
- **Fingerprint 失效判定。** Spec、repository 指示、issue 清單與每個 issue 的 SHA-256 digest，加上 branch/HEAD，決定快取的相依圖能否重用。穩定執行時，attempt 之間不會重讀 issue 內文；編輯、新增、刪除、重新命名 issue 或外部 commit 會觸發局部或完整重建。
- **全新 agent。** Worker 實作一個 issue。Verifier 為唯讀，檢查 commit ancestry、累積 diff、範圍、使用者變更的保留、驗收條件與各項檢查。Scheduler 負責重建相依圖。每個 agent 都使用全新對話，只拿到路徑，從不繼承上層對話歷史。
- **檔案系統 mailbox。** Agent 將結果與 log 寫在 `.orchestration/attempts/<id>/`、`scheduler/` 與 `integration/` 下，只回傳一個小型 envelope。Worker 與 verifier 的結果控制在約 1500 token 內，scheduler envelope 約 1000 token 內；完整 log 與 diff 留在磁碟上。
- **Write-ahead 轉換。** Issue Markdown 與 `state.json` 無法一起原子替換，因此每次 controller 寫入 issue 前，都先在 `state.json` 記錄 `pending_transition`（含寫入前後的 hash），再原子替換 issue 檔，最後一次寫入 `state.json` 完成轉換。復原時依 issue 的 hash 判斷該完成、重做，或視為外部變更；外部編輯絕不會被記成 controller 自己的變更。

因此 controller 的 context 隨 epoch 數量成長，而非隨累積的 feature 工作量成長。Context 隔離為必要條件；v1 刻意不做檔案系統隔離。請使用 host 原生的 fresh-agent 能力，並停用上層對話繼承。本套件不會在缺乏此能力的 host 上啟用委派。

## 重新進入（Re-entry）

自動重新進入只是最佳化，並非正確性的必要條件；重新呼叫 skill 永遠足夠。Skill 會使用 host 提供的最佳機制：

- **Thin driver 模式。** Runtime 能把每個 epoch 當成可再派出 agent 的全新隔離 agent 執行時，主對話只當作迴圈驅動器，每個 epoch 只保留約 300 token 的 envelope（`epoch`、`transition`、`next: continue|integrate|stop`、`report` 路徑）。
- **同對話模式。** 否則下一個 epoch 在同一對話中從步驟 1 重新自磁碟讀起。Host 發出 context 壓力或壓縮訊號，或用量限制中止執行時，skill 會讓位給使用者並提供續跑的呼叫方式。

無論哪種機制，每個 epoch 都從磁碟開始。記得的內容只是磁碟內容的快取。Driver 模式能否巢狀派出 agent 取決於 host。

## 狀態與復原

| Durable status | 意義 |
| --- | --- |
| `pending` | 尚未完成；只有在相依與安全檢查通過後才可被選取 |
| `in_progress` | 已持久化一個 attempt，正在進行或等待復原 |
| `blocked` | 證據指出前置條件、存取權、擁有權或重試次數的 blocker |
| `completed` | 驗收條件與實作證據已獨立驗證 |

`ready` 與 `invalid` 是排程分類，不是額外的狀態值。相依與風險分析由 scheduler 持久化為排序，決定每個 epoch 的下一個 issue；檔名數字只用於打破平手。每個 blocker 都附帶可觀察的解除條件（unblock predicate），每個 epoch 都會重新評估。其他安全的 ready 工作會繼續進行。

Issue 檔案保留 attempt 歷史、commit SHA、驗收證據與修正次數。本地 `.orchestration/` 證據保留 `state.json`、baseline、worker handle 與結果、驗證結果、controller 的 recovery lock，以及 integration 結果。請勿將這些本地快照 commit 進去；其中可能含有既有的使用者變更。在 repository 規則允許時，controller 可以另外 commit 自己簡潔的 issue metadata。

以相同的 feature 路徑再次呼叫 skill 即可續跑。它會先確認 worker 是否仍存活，再依持久化的 phase（`dispatch_pending`、`worker_running`、`worker_returned`、`verifying`、`verdict_accepted`）並對照 Git 完成中斷的 attempt；已寫出的 verification 會直接套用，不會重新驗證。遺失 controller 對話不會遺失資料。舊的時間戳記絕不代表可以重複執行工作。存活狀態或變更擁有權不確定時，會停止寫入直到釐清。必須透過 runtime lease 或保證單一 controller 的環境取得 checkout 的獨佔擁有權；feature 本地的 recovery lock 無法阻止不同 feature 同時被呼叫。本地復原需要原始的檔案系統證據，不保證磁碟遺失後仍可復原。Host 結束或用量限制阻止執行時，skill 無法繼續執行。

每個 issue 的初次 attempt 之外，最多允許兩次修正 attempt，次數跨重啟與最終 integration 持久計算。Verifier 回報 `incomplete` 或結果無法使用時，不算 worker 失敗，不消耗修正次數，只會以 `verifier_retries` 限制再派出一次全新 verifier。修正次數用盡會阻擋該 issue，直到獲得明確授權。Controller 絕不自行修正 worker 的程式碼。

## 完成與停止

完成需要：每個必要 issue 都已驗證為 completed、沒有進行中的 attempt 或未解決的 blocker、最終相關測試／型別檢查／lint／integration 審查全部通過、實作工作已 commit，且無關的使用者變更已保留。Repository 指示禁止 commit 時，視為明確記錄的例外。

最終 integration 由全新唯讀 verifier 執行。若發現某個 issue 造成的 regression，會重新開啟該 issue（保留修正次數），由新的修正 worker 在額度內處理，之後重跑 integration。不屬於任何單一 issue 的發現（缺漏、範圍、保留問題）會停下來回報，不會自行新增或擴大 ticket。

以下情況會停止並附上可行動的證據：沒有 ready 工作、spec 與 issue 實質矛盾、缺少必要的外部存取權、不可逆操作需要授權、writer 或 Git 擁有權無法釐清，或無法進行隔離委派。缺少輸入與相依循環都不能算作成功。不安全的 worker 部分編輯可能在共用 checkout 中阻擋其他工作。

此 skill 不會呼叫 `/loop`、`/implement` 或 `/implement-spec`，也不依賴其他實作 skill。未來的平行、worktree 或 tracker adapter 不在 v1 範圍內。

## 內容與驗證

- [SKILL.md](skills/engineering/issue-orchestrator/SKILL.md)：controller epoch、重新進入與硬性邊界。
- [Worker contract](skills/engineering/issue-orchestrator/references/worker-contract.md)：單一 issue 的執行方式與精簡結果 schema。
- [Scheduling](skills/engineering/issue-orchestrator/references/scheduling.md)：fingerprint、失效判定、全新 scheduler、相依圖與 ready frontier。
- [State machine](skills/engineering/issue-orchestrator/references/state-machine.md)：權威來源、`state.json`、mailbox 配置、write-ahead 轉換、epoch 復原與重試上限。
- [Verifier contract](skills/engineering/issue-orchestrator/references/verifier-contract.md)：唯讀的 attempt 與 integration 驗證，以及其結果 schema。
- [Verification](skills/engineering/issue-orchestrator/references/verification.md)：controller 如何接受驗證結果，以及最終 integration 關卡。
- [驗證紀錄](docs/validation.md)：安裝檢查、情境演練與 runtime 限制。

未內建任何 runtime 相依或可執行的 orchestration 服務。安裝相容性與 host 特定的執行能力分開驗證。Controller epoch 重構後的十個壓力情境只經過獨立 agent 的 decision trace 檢查，並未以真實 agent 實際執行；context 上限是 contract 目標，不是量測值。詳見驗證紀錄。

以 [MIT](LICENSE) 授權。
