# 第一階段 Spec：可展示的最小 FHIR Lakehouse

- 狀態：**草稿，待確認**（2026-10-16）
- 目標日期：2026-10-23
- 相關文件：[architecture.md](architecture.md)、[data-model.md](data-model.md)、[docs/adr/](adr/)、[GLOSSARY.md](../GLOSSARY.md)

## Problem Statement

我在轉職資料工程，需要一個能放進作品集的專案，證明我會處理真實世界資料管線的典型問題：巢狀的醫療資料格式、同一筆資料的多個版本、品質不一的資料、增量更新、可重現的結果，以及不綁定特定平台。

現在手上只有 Synthea 產生的一批 FHIR NDJSON 檔案，還沒有任何程式。這批資料有幾個「真實資料會有、但範例教學不會提」的問題：

- 沒有版本時間（`meta.lastUpdated`）。
- 只有一批，無法展示增量。
- 時間帶著產生資料那台電腦的時區。
- 「Condition」裡混了大量不是疾病的項目。

面試官要能在短時間內看懂這條管線做了什麼、為什麼這樣做，並且相信它是對的。

## Solution

建立一條 Bronze → Silver → Gold 的批次管線，處理 Patient、Encounter、Condition、Observation 四種資源：

- 用固定的種子產生 4 個批次（1 個初始批次 + 3 個每月更新批次），每批只把 Synthea 的模擬截止日往後延 30 天，讓同一個 id 真的有多個版本。
- **Bronze** 原樣保存每一行，記下批次和行號，筆數和來源檔完全對得上。
- **Silver** 把巢狀的 FHIR 攤平成表，保留代碼的系統和代碼值，解析資源之間的參照，每個 id 只保留最新版本，並且只寫入有變的資料。
- **品質規則**分成 `error`（擋下）和 `warn`（記錄但照常使用），每一筆違規都寫進 reject 表。
- **Gold** 提供病人維度、日期維度、就診事實表，以及「各年齡層前十大診斷」。
- 每一層都做筆數核對；同一批重跑，結果完全相同。
- 同一份程式能在本機和 Databricks 免費版上跑，差異只在 config。
- README、架構圖、ADR、LESSONS.md 說清楚每一個決定的理由。

## User Stories

**展示與理解**

1. 身為面試官，我想在 README 看到一張架構圖，以便一分鐘內理解資料怎麼從來源流到分析表。
2. 身為面試官，我想看到每個重大決策的 ADR，以便判斷作者是否考慮過其他選項。
3. 身為面試官，我想看到每一批每一層的筆數核對結果，以便相信沒有資料在中途憑空消失。
4. 身為面試官，我想看到 reject 表裡真實存在的違規資料，以便確認品質規則不是擺好看的。
5. 身為面試官，我想看到 LESSONS.md 記錄了踩過的坑，以便了解作者遇到問題時怎麼找原因。
6. 身為面試官，我想用一個指令就跑完測試，以便自己驗證程式是對的。
7. 身為面試官，我想看到「三種增量」各自做或不做的說明，以便確認作者懂增量的不同意思。

**資料產生**

8. 身為資料工程師，我想用一支腳本產生 4 個批次，以便隨時重現整套資料。
9. 身為資料工程師，我想固定 Synthea 的種子、醫師種子和參考日期，以便同樣的參數每次都產生完全相同的資料。
10. 身為資料工程師，我想讓更新批次是同一群病人模擬到更晚的日期，以便同一個 id 在不同批次有不同版本。
11. 身為資料工程師，我想讓每一批放在自己的資料夾，以便處理時不會混到其他批次的檔案。

**Bronze**

12. 身為資料工程師，我想把每一行 NDJSON 原封不動存進 Bronze，以便任何時候都能從 Bronze 重新產生 Silver。
13. 身為資料工程師，我想讓每一列記錄批次、來源檔和行號，以便追查任何一筆資料的出處。
14. 身為資料工程師，我想讓 Bronze 的列數和來源檔的行數對得上，以便確認載入沒有漏也沒有多。
15. 身為資料工程師，我想在同一種資源出現多個檔案時讓該批失敗，以便不會默默合併新舊兩次匯出。
16. 身為資料工程師，我想讓重跑同一批時整批取代，以便 Bronze 不會出現重複資料。
17. 身為資料工程師，我想讓格式錯誤的 JSON 也照樣存進 Bronze，以便筆數對得上，並由品質規則決定怎麼處理。

**Silver**

18. 身為分析師，我想要四種資源各有一張平的表，以便不用懂 FHIR 的巢狀結構也能查詢。
19. 身為分析師，我想保留每個代碼的系統和代碼值，以便知道一個代碼是 SNOMED 還是 LOINC。
20. 身為分析師，我想直接用 `patient_id`、`encounter_id` 關聯不同的表，以便不用自己解析參照字串。
21. 身為分析師，我想讓 Observation 的數值、單位、代碼值、文字值分在不同欄位，以便直接做數值運算。
22. 身為分析師，我想讓血壓這類有多個 component 的 Observation 拆成明細表，以便分別分析收縮壓和舒張壓。
23. 身為分析師，我想讓一筆 Observation 的所有 coding 都保留在明細表，以便用任何一個代碼查詢。
24. 身為分析師，我想讓日期欄位都是來源記錄的日期，以便清晨的就診不會被算到前一天。
25. 身為資料工程師，我想讓每個 id 在 Silver 只有一列最新版本，以便下游不用自己挑版本。
26. 身為資料工程師，我想讓有效更新時間優先採用資料本身的 `lastUpdated`，以便以後接上真實來源時不用改規則。
27. 身為資料工程師，我想讓同一批裡同一個 id 出現多次時，固定由行號較後的勝出，以便結果不會隨機。
28. 身為資料工程師，我想讓 Silver 只寫入新增和有變的資料，以便展示寫入增量。
29. 身為資料工程師，我想讓比 Silver 現有版本舊的資料不寫入，以便晚到的舊資料不會蓋掉新資料。
30. 身為資料工程師，我想讓「這一批沒有出現」的 id 繼續留在 Silver，以便不會把掉出匯出範圍誤當成刪除。
31. 身為學習 Python 的工程師，我想自己實作 Patient 和 Encounter 的攤平函式，以便練習巢狀欄位和陣列的處理。
32. 身為學習 Python 的工程師，我想自己實作「選最新版本」的函式，以便練習 window function 和多欄位排序。
33. 身為資料工程師，我想讓 Silver 的 Patient 不包含 SSN、電話、街道地址，以便養成處理醫療個資的習慣。

**品質**

34. 身為品質負責人，我想讓每種資源至少有三條品質規則，以便涵蓋必填欄位、參照和代碼格式。
35. 身為品質負責人，我想讓每條規則有固定代號和嚴重度，以便在 reject 表裡依規則統計。
36. 身為品質負責人，我想讓 `error` 級的違規資料不進 Silver，以便下游不會拿到斷掉的參照。
37. 身為品質負責人，我想讓 `warn` 級的違規資料照常進 Silver 並留下紀錄，以便不會因為小問題丟掉有用的資料。
38. 身為品質負責人，我想讓一筆資料違反幾條規則就有幾筆 reject 紀錄，以便知道每一筆的完整問題。
39. 身為品質負責人，我想讓缺 id 的資料也能在 reject 表找到，以便追查到來源檔的哪一行。
40. 身為品質負責人，我想讓指向被擋下資料的參照也被擋下，以便 Silver 裡沒有任何斷掉的關聯。

**Gold**

41. 身為分析師，我想要一張病人維度表，以便依性別、種族、州來切分就診。
42. 身為分析師，我想要一張連續的日期維度表，以便依年、季、月、星期分析，沒有就診的日子也不會缺。
43. 身為分析師，我想要一張就診事實表，包含時長和診斷數，以便分析就診的量和型態。
44. 身為分析師，我想查到各年齡層的前十大診斷，以便了解不同年齡層的主要疾病。
45. 身為分析師，我想讓「診斷」只包含疾病，不包含「壓力」「全職工作」這類社會因素，以便排名有臨床意義。
46. 身為分析師，我想讓年齡用發病當下的年齡，以便已故病人的診斷也能正確歸類。
47. 身為分析師，我想讓同分時的排名固定，以便每次查詢的結果都一樣。

**一致性與維運**

48. 身為資料工程師，我想讓同一批重跑之後，除了稽核欄位，所有表都和上次完全相同，以便放心重跑。
49. 身為資料工程師，我想讓在後面的批次之後重跑前面的批次，不會把資料改回舊版，以便重跑任何一批都安全。
50. 身為資料工程師，我想讓每一批的狀態記在控制表，以便知道哪一批成功、哪一批失敗。
51. 身為資料工程師，我想讓筆數核對不通過時，該批記為失敗，以便問題不會被忽略。
52. 身為資料工程師，我想在修正攤平邏輯後重跑同一批就能套用新邏輯，以便修 bug 不用整個重建。

**平台**

53. 身為資料工程師，我想讓程式只用邏輯表名，以便同一份程式在本機和 Databricks 上都能跑。
54. 身為資料工程師，我想讓所有平台相關的設定都集中在 config，以便換平台只要換一份 config。
55. 身為資料工程師，我想讓 Spark 的時區在所有環境都固定為 UTC，以便兩個平台算出相同的結果。
56. 身為資料工程師，我想在 Databricks 免費版用同一份程式跑通一批，以便證明程式真的不綁定平台。

## Implementation Decisions

### 模組

| 模組 | 負責 | 介面 |
|---|---|---|
| 設定與表位置解析 | 我 | 讀取指定環境的 config；把邏輯表名解析成實際的讀寫位置；建立或取得 Spark session |
| 批次登記 | 我 | 驗證批次資料夾、讀 Synthea metadata、寫入 `ops.batch` |
| Bronze 載入 | 我 | 讀一種資源的 NDJSON，加上簿記欄位，整批取代寫入 |
| 解析 | 我 | 每種資源一份明確的 schema；把 `raw` 解析成 `resource` 結構，算出有效更新時間 |
| 共用小函式 | 我 | 從參照字串取出 id（格式不對就回傳 null）、取記錄日期、從多個 coding 選出主要代碼 |
| 攤平：Patient、Encounter | **你** | 見下方〈攤平函式的介面〉 |
| 攤平：Condition、Observation | 我 | 同上；Observation 另外產生 coding 和 component 明細 |
| 品質規則 | 我 | 規則以資料定義（代號、嚴重度、資源、條件）；輸入攤平後的資料，輸出「通過的資料」和「reject 紀錄」 |
| 選最新版本 | **你** | 見下方〈你負責的函式〉 |
| MERGE | 我 | 依 ADR-0005 把最新版本寫入 Silver，回傳各去向的筆數；找出同批被淘汰的版本，記 `COM-003` |
| Gold | 我 | Spark SQL，從 Silver 全量重建四張 Gold 表 |
| 核對 | 我 | 寫入 `ops.reconciliation`，任何一項不通過就讓批次失敗 |
| 入口 | 我 | 「處理一批」：輸入環境、`batch_id`、來源資料夾 |
| 資料產生腳本 | 我 | 產生 4 個批次，參數見〈Further Notes〉 |

### 你負責的函式

三個函式都由你實作，我提供函式簽名、docstring 和測試。三個都是純函式：輸入和輸出都是 DataFrame，不讀寫任何表。

**攤平：Patient、Encounter**

- **輸入**：一列是一筆 Bronze 資料；`resource` 欄位已經用明確 schema 解析成巢狀結構；另外帶著簿記欄位 `_batch_id`、`_batch_seq`、`_line_no`、`_effective_last_updated`。
- **輸出**：簿記欄位原樣保留，加上 [data-model.md](data-model.md) 列出的業務欄位。
- **只做攤平**：不去重、不檢查品質。
- **輸入幾列就輸出幾列**：欄位缺值就填 null，不能丟掉任何一列。
- Condition、Observation 的攤平函式由我實作，遵守同樣的介面。

**選最新版本**

- 四種資源共用同一個函式，參數是 DataFrame 和 id 欄位的名稱（例如 `patient_id`）。
- **輸入**：已經通過 `error` 級規則的資料，包含攤平後的業務欄位和簿記欄位。同一個 id 可能有多列；id 不會是 null（缺 id 的已經被 `COM-002` 擋下）。
- **輸出**：每個 id 只剩一列，也就是最新版本；欄位和輸入完全相同。
- **怎麼判斷最新**：依序比較 `_effective_last_updated`、`_batch_seq`、`_line_no`，大者為新（[ADR-0001](adr/0001-version-ordering.md)）。這三個欄位合起來不會重複，所以結果是固定的。
- **範圍**：只在輸入的資料之間挑選，不讀 Silver。和 Silver 現有版本比較，是 MERGE 的工作。
- 被淘汰的版本由我在 MERGE 那一步找出來，記 `COM-003`。

### 版本與增量

- 版本比較鍵：（有效更新時間, 批次序號, 行號），依序比較，大者為新（[ADR-0001](adr/0001-version-ordering.md)）。
- 有效更新時間 = `meta.lastUpdated`，缺值時用批次的資料截止日（Synthea metadata 的 `endTime`，以當天 00:00 UTC 表示）。資料截止日是批次層級的時間，同一批每筆資料都一樣，不是每筆資料各自的更新時間；判斷內容有沒有變，看內容雜湊值。
- 處理順序：先檢查品質，再從通過的版本中選最新（[ADR-0004](adr/0004-reject-table-and-severity.md)）。
- 同一批中被淘汰的重複版本，記一筆 `COM-003`（warn）。
- Silver 的寫入方式：新的 id 就新增；版本較新或相同、而且內容雜湊值不同就更新；其他不寫（[ADR-0005](adr/0005-incremental-write.md)）。
- 內容雜湊值只涵蓋業務欄位。
- Gold 每批全量重建。
- 未出現不是刪除；第一階段不處理刪除。

### 批次

- 一批 = 一次 Synthea 執行的完整輸出資料夾。
- `batch_id` 格式固定為 `batch_NNN`，批次序號就是 `NNN`，由執行參數指定，不從執行時間產生。
- 登記批次時，從 metadata 讀資料截止日；缺值或格式不是 `YYYYMMDD`，該批失敗。
- 同一批內，四種資源依序處理：Patient → Encounter → Condition、Observation。每種處理完就先 MERGE 進 Silver，下一種檢查參照時才查得到。
- 「參照的資料存在」查的是 Silver 現有資料，其中已經包含同一批先處理完的資源。
- 重跑同一批時，Bronze 和 reject 紀錄整批取代；Silver 依版本規則 MERGE；Gold 全量重建。

### 品質規則

| 代號 | 資源 | 規則 | 嚴重度 |
|---|---|---|---|
| `COM-001` | 全部 | JSON 解析失敗，或 `resourceType` 和檔案不符 | error |
| `COM-002` | 全部 | `id` 缺值或空字串 | error |
| `COM-003` | 全部 | 同一批內同一個 id 出現多次（被淘汰的版本） | warn |
| `PAT-001` | Patient | `birthDate` 缺值 | warn |
| `PAT-002` | Patient | `gender` 不是 male、female、other、unknown 之一 | warn |
| `PAT-003` | Patient | 死亡日早於出生日 | warn |
| `ENC-001` | Encounter | 病人參照的格式不是 `Patient/<id>` | error |
| `ENC-002` | Encounter | 參照的病人不存在 | error |
| `ENC-003` | Encounter | 缺開始時間 | error |
| `ENC-004` | Encounter | 結束時間早於開始時間 | warn |
| `CON-001` | Condition | Encounter 參照的格式不是 `Encounter/<id>` | error |
| `CON-002` | Condition | 參照的 Encounter 不存在 | error |
| `CON-003` | Condition | SNOMED 代碼不是 6 到 18 位數字 | warn |
| `CON-004` | Condition | Condition 的病人和其 Encounter 的病人不同 | warn |
| `OBS-001` | Observation | Encounter 參照的格式不是 `Encounter/<id>` | error |
| `OBS-002` | Observation | 參照的 Encounter 不存在 | error |
| `OBS-003` | Observation | LOINC 代碼不符合「1 到 5 位數字、連字號、1 位檢查碼」 | warn |
| `OBS-004` | Observation | 沒有值，也沒有 component | warn |

代碼格式只檢查格式，不檢查 LOINC 的檢查碼是否正確，也不查代碼是否真的存在。

### 攤平的規則

- **主要代碼**：優先選該資源預期的代碼系統（Condition 用 SNOMED，Observation 用 LOINC），同系統有多個就取第一個；都沒有就取第一個 coding。
- **Patient 名字**：優先取 `use = official`，沒有就取第一個；多個名字以空格串接。
- **Patient 種族、族裔**：取 US Core extension 裡的 `ombCategory`。
- **Patient 婚姻狀態**：只存代碼和說明。說明取 coding 的 `display`，沒有就取 `text`。不另建代碼表。
- **Encounter 原因**：取第一個 `reasonCode`。
- **Observation 的值**：依 `value[x]` 的型態，填入對應的值欄位，並記錄 `value_type`；只有 component 的，`value_type = component`。
- **Condition 語意標籤**：取顯示名稱結尾括號內的文字。這是從文字推斷的權宜做法，因為第一階段沒有 SNOMED 的階層資料。
- **日期**：一律是記錄日期，取時間字串的前 10 個字元（[ADR-0008](adr/0008-timezone-and-dates.md)）。

### 各年齡層前十大診斷

- 診斷 = `semantic_tag` 為 `disorder` 的 Condition。在目前的資料中只佔 33%，排名前面的多半是 `finding` 和 `situation`，例如「Medication review due」8,160 筆、「Stress」3,052 筆。
- 年齡 = 發病年齡（足歲）。沒有出生日的病人不計入。
- 年齡層：`0-17`、`18-44`、`45-64`、`65+`。
- 計數 = 不重複病人數。
- 排名：病人數由多到少，同分時依代碼由小到大，每個年齡層固定取 10 名。

### 平台

- 程式只用邏輯表名，實際位置由 config 決定：本機是 Delta 路徑，Databricks 是 Unity Catalog 受管理表（[ADR-0007](adr/0007-platform-portability.md)）。
- 每個環境一份 TOML config。
- 所有環境的 Spark 時區固定 UTC（[ADR-0008](adr/0008-timezone-and-dates.md)）。
- 不使用 RDD 和 SparkContext。
- 本機和 Databricks serverless 都用 Python 3.12。
- Databricks 用 Git 資料夾連 repo，由 notebook 呼叫同一個入口；來源資料放 Unity Catalog Volume。

## Testing Decisions

### 好的測試長什麼樣

- 只測外部行為：給定輸入資料，檢查輸出的表內容和筆數，不檢查內部用了哪些 DataFrame 操作。
- 每個測試的輸入資料要小到一眼看得完，期望值直接寫在測試裡。
- 每條品質規則至少兩個案例：一筆會觸發、一筆不會觸發。

### 兩個測試接縫

1. **單一函式**：用手工的小 DataFrame 測試。這一層是你實作時的主要依據。
   - **攤平**（Patient、Encounter、Condition、Observation）：
     - 正常資料攤平後的每個欄位值
     - 缺值的欄位變成 null，而且列數不變
     - 多個名字、多個 coding、多個 component 時取哪一個
     - 參照格式錯誤時 id 為 null
     - 記錄日期不受時區影響
   - **選最新版本**：
     - `lastUpdated` 較晚的勝出，即使它的批次序號比較小
     - `lastUpdated` 相同時，批次序號大的勝出
     - 同一批內，行號大的勝出
     - 每個 id 只輸出一列，不同 id 互不影響
     - 輸出的欄位和輸入完全相同
2. **整批處理**：在 `tests/fixtures/` 準備 2 到 3 個手工小批次，每個都有 metadata 和四種資源的 NDJSON。測試時對暫存資料夾跑「處理一批」，再檢查所有表。
   - **多版本**：同一個 id 在不同批次、有 `lastUpdated` 和沒有 `lastUpdated` 的情況
   - **晚到的舊版本**：後面的批次帶著比較舊的 `lastUpdated`，不能蓋掉新版本
   - **重複 id**：同一批內同一個 id 出現兩次
   - **缺值**：缺 id、缺出生日、缺開始時間
   - **代碼格式錯誤**：SNOMED 有字母、LOINC 格式不對
   - **斷掉的參照**：找不到病人、找不到 Encounter，以及連鎖擋下
   - **格式錯誤的 JSON**
   - **批次本身有問題**：同一種資源有多個檔案、metadata 缺少資料截止日
   - **筆數核對**：每一層的核對都通過；故意讓核對失敗時，批次記為 `failed`
   - **重跑一致**：同一批跑兩次，除稽核欄位外所有表完全相同
   - **順序無關**：依 1、2、3 和 1、3、2 的順序處理，Silver 結果相同
   - **未出現不是刪除**：後面的批次沒有某個 id，它仍留在 Silver

### 其他

- 所有測試共用同一個 Spark session（啟動一次約 10 秒）。
- 測試不依賴完整的 Synthea 資料，`uv run pytest` 在乾淨的 clone 上就能跑。
- 目前 repo 還沒有任何測試，沒有前例可循；第一批測試就是之後的範本。

## 驗收標準

### 1. Bronze

- [ ] 四種資源各有一張 Bronze 表，結構符合 [data-model.md](data-model.md)。
- [ ] 每一批、每種資源的 Bronze 列數 = 來源檔行數，`ops.reconciliation` 記錄為通過。
- [ ] `raw` 和來源檔的那一行逐字相同；`_line_no` 和實際行號相符。
- [ ] 每一列都有 `_batch_id`、`_batch_seq`、`_source_file`、`_line_no`、`_loaded_at`。
- [ ] 同一種資源有多個檔案時，該批失敗，並說明是哪些檔案。
- [ ] metadata 缺少資料截止日，或格式不對時，該批失敗。
- [ ] 重跑同一批，Bronze 的列數不變。
- [ ] 格式錯誤的 JSON 照樣存進 Bronze，`resource_id` 為 null。

### 2. Silver

- [ ] 有 `patient`、`encounter`、`condition`、`observation` 四張主表，以及 `observation_coding`、`observation_component` 兩張明細表，結構符合 [data-model.md](data-model.md)。
- [ ] 每個主表的主鍵不重複。
- [ ] 每個代碼都保留系統和代碼值；Observation 的所有 coding 都在明細表中。
- [ ] Encounter 的 `patient_id` 都能在 `silver.patient` 找到；Condition 和 Observation 的 `encounter_id` 都能在 `silver.encounter` 找到。
- [ ] 同一個 id 有多個版本時，Silver 中是依 ADR-0001 判斷的最新版本。
- [ ] 攤平函式的輸出列數 = 輸入列數。
- [ ] 所有日期欄位都是記錄日期。
- [ ] `silver.patient` 不包含 SSN、駕照、護照、電話、街道地址。
- [ ] Patient、Encounter 的攤平函式和「選最新版本」函式由你實作，並通過測試。

### 3. 品質

- [ ] 上方〈品質規則〉表中的 18 條規則都已實作，每種資源至少三條（不含共用規則）。
- [ ] 每條規則都有「會觸發」和「不會觸發」的測試。
- [ ] `error` 級違規的資料不在 Silver；`warn` 級違規的資料在 Silver，也在 reject 表。
- [ ] 一筆資料違反幾條規則，reject 表就有幾筆紀錄；主鍵為（規則, 批次, 資源類型, 行號）。
- [ ] 重跑同一批，reject 紀錄的筆數不變。
- [ ] 用完整資料跑 `batch_001`：`error` 級規則都是 0 筆；`OBS-003` 的筆數和實際 LOINC 格式錯誤的 Observation 數一致（舊資料是 1,288 筆，重新產生後以實際數字為準）。

### 4. Gold

- [ ] `dim_patient`、`dim_date`、`fact_encounter`、`top_diagnoses_by_age_band` 四張表的結構符合 [data-model.md](data-model.md)。
- [ ] `dim_patient` 列數 = `silver.patient` 列數；`fact_encounter` 列數 = `silver.encounter` 列數。
- [ ] `dim_date` 從最小到最大日期連續不跳號，事實表的日期鍵都找得到。
- [ ] `fact_encounter` 的 `patient_id` 都能在 `dim_patient` 找到。
- [ ] 前十大診斷：4 個年齡層，每層最多 10 列；只包含診斷；用的是發病年齡和不重複病人數；同分時的排名固定。
- [ ] Gold 的 SQL 放在 `sql/`。

### 5. 增量

- [ ] 4 個批次依序處理完，每一批的 Silver 核對都通過，並記錄新增、更新、未變、過期、同批重複、擋下的筆數。
- [ ] `batch_002` 之後的批次，Silver 有被更新的列，也有未變的列。
- [ ] 同一批重跑，除稽核欄位外，所有表的內容完全相同。
- [ ] 在 `batch_004` 之後重跑 `batch_002`，Silver 沒有任何一列被改變。
- [ ] 某個 id 在後面的批次沒有出現，仍留在 Silver。
- [ ] 文件說明三種增量各自做或不做，以及原因。

### 6. 文件

- [ ] README：專案目的、架構圖、怎麼產生資料、怎麼執行、怎麼測試、每一批的筆數和 reject 數。
- [ ] [architecture.md](architecture.md)、[data-model.md](data-model.md)、[docs/adr/](adr/) 和實作一致。
- [ ] LESSONS.md 記錄這一階段踩到的坑。

### 限制

- [ ] 程式裡沒有寫死的路徑，所有平台相關設定都在 config。
- [ ] 在 Databricks 免費版，用同一份程式跑通至少一批（可以用小批次）。
- [ ] `uv run pytest` 和 `uv run ruff check .` 都通過，測試只用 `tests/fixtures/`。

## Out of Scope

- **刪除**：第二階段用刪除清單或 `entered-in-error` 模擬。
- 四種以外的 FHIR 資源（Claim、Procedure、MedicationRequest 等）。
- 機構、醫師的維度；Encounter 的機構參照只保留原始字串。
- Silver 和 Gold 的歷史版本（SCD2）。歷史版本留在 Bronze。
- 抽取增量。
- 參照的資料晚一批才到時，自動重新處理之前被擋下的資料。
- LOINC 檢查碼驗證、代碼是否真的存在、SNOMED 階層。
- 多個來源系統的 id 衝突。
- Microsoft Fabric。
- 排程、串流、CI、效能調校。

## Further Notes

### 資料產生

- 四批都用 `-p 1000 -s 42 -cs 42 -r 20261013`，`-e` 依序是 `20261013`、`20261112`、`20261212`、`20270111`（[ADR-0002](adr/0002-update-batches-by-end-date.md)）。
- 原本規劃的快照做法，實測 1000 人時 Java 記憶體不足，已放棄。
- 每批各自獨立執行，約 1 分多鐘；每批放在自己的資料夾，資料夾不進 Git。
- 第一次產生完整資料後，要確認每批都有新增和更新的資料（2 人的實驗中 30 天內沒有新增）。

### 時程

原排程是 10/15 Bronze、10/16 Patient、10/17 Encounter。今天是 10/16，Bronze 還沒開始，拆票時需要重新排日期。

### 已知風險

- **行號**：Databricks serverless 不支援 RDD，不能用常見的 `zipWithIndex` 取行號。Bronze 的票要先驗證取行號的方法，並用測試確認行號和實際行號相符。
- **Mermaid 圖**：架構圖沒有在本機轉成圖片驗證過，要在 GitHub 上確認能正常顯示。
