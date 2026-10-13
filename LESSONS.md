# LESSONS

踩過的坑。每條寫：現象、原因、做法。

## 2026-10-14 Synthea bulk 輸出的檔名不一定是 `<Resource>.ndjson`

- 現象：`output/fhir/` 裡 Location、Organization、Practitioner、PractitionerRole 的檔名帶數字後綴，例如 `Location.1791905149569.ndjson`；其他資源沒有。
- 原因：這四種是 Synthea 另外匯出的機構／醫事人員資源，後綴是匯出當下的毫秒時間戳。在同一個目錄重跑 Synthea、沒先清空的話，會多出一組新時間戳的檔案，和舊檔並存。
- 做法：
  - ingest 用 `<Resource>*.ndjson` 匹配，不寫死檔名。
  - `output/fhir/parameters.json` 列出了「資源類型 → 檔名」，可以當作這批的清單來核對檔案數。
  - 重新產生資料前先清空 `output/`；ingest 遇到同一種資源有多個檔案時要報錯，不要默默合併。
