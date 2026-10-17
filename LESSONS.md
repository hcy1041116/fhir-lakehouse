# LESSONS

踩過的坑。每條寫：現象、原因、做法。

## 2026-10-14 Synthea bulk 輸出的檔名不一定是 `<Resource>.ndjson`

- 現象：`output/fhir/` 裡 Location、Organization、Practitioner、PractitionerRole 的檔名帶數字後綴，例如 `Location.1791905149569.ndjson`；其他資源沒有。
- 原因：這四種是 Synthea 另外匯出的機構／醫事人員資源，後綴是**參考日期**（`-r`）的毫秒時間戳。沒指定 `-r` 時，參考日期就是執行當下，所以看起來像匯出時間（2026-10-16 更正：固定 `-r 20261013` 時，後綴是 `1791849600000`，即 2026-10-13 00:00 UTC）。在同一個目錄重跑 Synthea、沒先清空、參考日期又不同的話，會多出一組新後綴的檔案，和舊檔並存。
- 做法：
  - ingest 用 `<Resource>*.ndjson` 匹配，不寫死檔名。
  - `output/fhir/parameters.json` 列出了「資源類型 → 檔名」，可以當作這批的清單來核對檔案數。
  - 重新產生資料前先清空 `output/`；ingest 遇到同一種資源有多個檔案時要報錯，不要默默合併。

## 2026-10-15 Synthea 的資料沒有 `meta.lastUpdated`

- 現象：Patient、Encounter、Condition、Observation 共 683,514 筆，`meta` 裡只有 `profile`，沒有 `lastUpdated`，也沒有 `versionId`。AGENTS.md「以 `meta.lastUpdated` 取最新」的規則，第一步永遠比不出結果。
- 原因：Synthea 是模擬器，不是 FHIR 伺服器；`lastUpdated` 通常是伺服器在儲存資源時才填上的。
- 做法：有效更新時間缺值時，改用批次的資料截止日，再依序比批次序號、行號（ADR-0001）。資料截止日是批次層級的時間，不是每筆資料的更新時間。「`lastUpdated` 不同」的情況用 `tests/fixtures/` 的手工資料測。

## 2026-10-15 換 Synthea 種子產不出「同一個 id 的新版本」

- 現象：用 3 個病人實驗，`-s 42` 和 `-s 43` 兩批，四種資源的 id 完全沒有重疊。拿來測增量，只會測到新增，測不到更新。
- 原因：Synthea 的 id 是從種子推導出來的，換種子就是一群全新的病人。
- 做法：種子和參考日期都固定，每批只把模擬截止日 `-e` 往後延（ADR-0002）。原本規劃用快照接續，但 1000 人時記憶體不足，見下方「Synthea 存快照會把記憶體吃光」。

## 2026-10-15 Synthea 同樣的 `-s` 重跑，Encounter 還是不一樣

- 現象：兩次都用 `-s 42 -r 20261013`，Patient、Condition、Observation 完全相同，Encounter 的 `participant`（醫師）不同。
- 原因：醫師種子 `-cs` 沒指定時，預設用執行當下的時間。另外 `-r`（參考日期）沒指定時，預設是今天。
- 做法：產生資料時一定同時固定 `-s`、`-cs`、`-r`。固定之後，同樣參數的輸出逐字相同。

## 2026-10-15 用快照接續時，Synthea metadata 的 `endTime` 不可靠

- 現象：連續三批快照接續（`-i` 載入、`-t 30`）的實驗中，第三批 metadata 的 `endTime` 是 `20180614`，比第一批還早。
- 原因：不明，可能是 Synthea 載入快照時沒有正確更新這個欄位。
- 做法：不用快照做法。改用 `-e` 之後實測，`endTime` 等於 `-e` 的值，可以當作資料截止日；登記批次時仍要檢查它存在、格式是 `YYYYMMDD`。

## 2026-10-15 時間字串帶著產資料那台電腦的時區

- 現象：時間字串大多是 `+08:00`，有 334 筆 Encounter 是 `+09:00`。本機 Spark 的 session 時區預設是 `Asia/Taipei`，Databricks 預設是 UTC。`2018-03-30T04:31:33+08:00` 轉成 timestamp 再取日期，UTC 下是 3/29，台北時區下是 3/30。
- 原因：Synthea 用 JVM 的預設時區輸出時間；`+09:00` 是台灣早年實施夏令時間的年份。Spark 從 timestamp 取日期時，會用 session 時區換算。
- 做法：所有環境的 session 時區固定為 UTC；日期欄位直接取字串前 10 個字元（記錄日期），不從 timestamp 換算（ADR-0008）。

## 2026-10-15 Condition 不等於診斷

- 現象：Condition 前幾名是「Medication review due (situation)」8,160 筆、「Stress (finding)」3,052 筆、「Full-time employment (finding)」2,842 筆。直接算「前十大診斷」，名單會被這些非疾病的項目佔滿。
- 原因：Synthea 把社會因素、健康狀況評估也輸出成 Condition。顯示名稱結尾括號裡的 SNOMED 語意標籤可以分辨：`disorder` 只佔 33%，其餘是 `finding`、`situation` 等。
- 做法：「診斷」定義為語意標籤為 `disorder` 的 Condition（GLOSSARY.md）。從顯示名稱推斷語意標籤是權宜做法，因為第一階段沒有 SNOMED 的階層資料。

## 2026-10-16 Synthea 存快照會把記憶體吃光

- 現象：`-p 1000` 加上 `-u` 存快照，執行 26 分鐘只完成約 295 人，stderr 一直出現 `OutOfMemoryError: Java heap space`，CPU 500% 以上卻幾乎沒有進度，連一般的停止訊號都沒反應，最後只能強制結束。不存快照時，同樣 1000 人只要 73 秒。
- 原因：不存快照時，Synthea 模擬完一個人就寫出病歷、從記憶體丟掉。存快照時，所有病人的模擬狀態都要留在記憶體裡，等全部模擬完才一起寫進同一個檔案。實測 58 人就用掉約 1.9GB；Java 預設最多只用總記憶體的 1/4（這台約 2GB），1000 人遠遠不夠，調高上限也超過 8GB。
- 做法：
  - 不用快照，改用 `-e` 延長模擬截止日（ADR-0002）。
  - 跑大量資料前，先用少量人數試跑，看 `/usr/bin/time` 的 `maxRSS` 估算記憶體。
  - 長時間沒進度時，先看 stderr 有沒有 `OutOfMemoryError`，不要一直等。

## 2026-10-17 Synthea 保留全部歷史又設未來的截止日，會拒絕執行

- 現象：加上 `--exporter.years_of_history=0`、`-e 20261112`（在執行當天之後），Synthea 回傳 exit code 0，卻沒有產生任何檔案。log 裡寫著 `Error: the specified end time is further in the future than the number of years of export history`。
- 原因：Synthea 會檢查「截止日減去匯出年限」是否還在未來，是的話代表第一筆匯出的資料就在未來，它認為參數設錯而拒絕執行。`years_of_history=0` 時，截止日只要在未來就會被擋。
- 做法：
  - 產生資料時把所有截止日放在過去（ADR-0002 改從 `-r 20251013` 開始），就不需要強制執行。
  - 真的需要未來的日期時，把 `-e` 換成 `-E` 強制執行。
  - Synthea 失敗時 exit code 可能還是 0，產生資料的腳本要檢查輸出檔案是否存在，不能只看 exit code。
