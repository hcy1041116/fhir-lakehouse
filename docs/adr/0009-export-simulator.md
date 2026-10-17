# 0009 在 Synthea 和 Bronze 之間加入匯出模擬器

- 狀態：accepted（2026-10-17）

## 背景

真實的 FHIR 分析平台，通常透過 FHIR Bulk Data 的 `$export` 從 FHIR 伺服器取資料，並用 `_since` 參數做增量。Bulk Data 規格（3.0.0）的定義：

> Resources will be included in the response if their state has changed after the supplied time (e.g., if `Resource.meta.lastUpdated` is later than the supplied `_since` time).

每次匯出完成時，伺服器會回傳一份**匯出清單**（manifest），其中 `transactionTime` 表示「這次查詢執行的時間」，匯出的資料不應包含在這之後才修改的資源。規格建議的增量流程是：用戶端保留這次的 `transactionTime`，下一次匯出時當作 `_since` 傳回去。

Synthea 沒有這些能力：

- 每次執行都是**完整匯出**，沒辦法只給「上次之後有變的」。
- 683,514 筆資料都沒有 `meta.lastUpdated`。
- 沒有匯出清單。

在這個前提下，原本的設計只能「抽取增量不做」：每一批都要讀、解析、檢查整份資料，計算量和完整資料成正比；版本比較也只能用批次層級的時間代替 `lastUpdated`。這和真實平台的樣子差很多。

## 選項

- **A. 維持現狀**：管線直接讀 Synthea 的完整匯出，抽取增量不做。
- **B. 在管線裡做差異比對**：Bronze 收完整匯出，由管線自己比對前後兩批，只把有變的資料送進 Silver。
- **C. 匯出模擬器**：在管線**外面**加一個程式，扮演 FHIR 伺服器。它比對前後兩份完整匯出，只輸出新增和有變的資料，替每筆資料補上 `meta.lastUpdated`，並附上 Bulk Data 格式的匯出清單。

## 決定

選 C。管線只讀匯出模擬器的輸出，不直接讀 Synthea。

**匯出模擬器的行為：**

| | 初始匯出（`batch_001`） | 增量匯出（`batch_002` 之後） |
|---|---|---|
| 輸入 | 本批的完整匯出 | 本批和上一批的完整匯出 |
| 輸出哪些資料 | 全部 | 新增的 id，以及兩批都有但內容不同的 id |
| 內容相同的資料 | — | 不輸出 |
| 上一批有、本批沒有的 id | — | 不輸出，也**不視為刪除**；筆數記在匯出報告中（保留全部歷史時預期為 0） |
| `meta.lastUpdated` | 本批資料截止日 00:00:00Z | 本批資料截止日 00:00:00Z |
| `_since` | 無 | 上一批的 `transactionTime` |

- 除了補上 `meta.lastUpdated`，每筆資料的其他內容原封不動。
- 只處理 Patient、Encounter、Condition、Observation 四種資源。
- 兩份完整匯出用同一個 id 的原始 JSON 逐字比較，判斷內容有沒有變。

**匯出清單**依照 Bulk Data 3.0.0 的完成回應格式：

| 欄位 | 值 |
|---|---|
| `transactionTime` | 本批資料截止日 00:00:00Z |
| `request` | 模擬的請求網址，增量匯出時含 `_since` |
| `requiresAccessToken` | `false` |
| `output` | 每種資源一個檔案，含 `type`、`url`、`count` |
| `deleted` | 第一階段一律為空 |
| `error` | 空 |

`url` 用相對檔名，不是規格要求的絕對網址，因為檔案在本機。這是唯一刻意偏離規格的地方。

## 理由

- **貼近真實**：管線收到的資料和真實 FHIR 平台一樣，只有變動的部分、每筆都有 `lastUpdated`、附有匯出清單。Bronze 之後的程式碼完全不需要知道來源是 Synthea，以後接上真的 FHIR 伺服器，只要換掉模擬器。
- **模擬器在管線外面，所以不違反「Bronze 原樣保存」**：補 `lastUpdated` 是「來源系統」的行為，不是管線在改資料。從管線的角度看，Bronze 仍然原封不動保存來源給的東西（ADR-0003）。ADR-0001 拒絕「自己補 `lastUpdated`」，指的是在管線**裡面**修改資料，兩者不衝突。
- **B 會把來源的限制帶進管線**：管線必須拿到上一批的完整內容才能比對。真實的 Bulk Data 來源不會給完整匯出，B 的程式碼以後就用不到了。
- **A 在資料變大時不可行**：抽取和計算都是全量，傳輸、儲存、計算量都和完整資料成正比。
- **`lastUpdated` 只能精確到批次**：Synthea 不會告訴我們某筆資料是在這 30 天內的哪一天改的，所以統一用本批的資料截止日。這仍然是批次層級的時間，不是每筆資料真正的修改時間（ADR-0001）。

## 後果

- **三種增量都做了**：抽取增量由模擬器模擬；計算增量從 `batch_002` 起只和變動量成正比；寫入增量照 ADR-0005。
- **「未出現」在增量匯出中是常態**：沒有變動的資料本來就不會出現，這正是 `_since` 的意思。
- **版本比較**：每筆資料都有 `lastUpdated`，ADR-0001 的「缺值時用資料截止日」只剩防呆作用，例如測試資料沒有 `lastUpdated` 的情況。
- **批次控制改讀匯出清單**：批次的資料截止日、`_since`、每個檔案的預期筆數，都從匯出清單取得，不再讀 Synthea 的 metadata。Bronze 的筆數核對多一層：清單上的 `count` = 檔案行數 = Bronze 列數。
- **完整匯出要保留**：產生增量匯出需要上一批的完整匯出。重新產生某一批的完整匯出時，這一批和下一批的增量匯出都要重新產生。
- **刪除**：匯出清單的 `deleted` 欄位已經預留，第二階段可以用刪除清單或 `entered-in-error` 模擬。
- 匯出模擬器有自己的測試：用兩份手工的小型完整匯出，檢查新增、有變、未變、未出現四種情況的輸出。

## 參考

- [FHIR Bulk Data Access IG](https://hl7.org/fhir/uv/bulkdata/export.html)，`_since` 參數與完成回應的欄位；原始檔見 [HL7/bulk-data 3.0.0](https://github.com/HL7/bulk-data/blob/3.0.0/input/pagecontent/export.md)。
