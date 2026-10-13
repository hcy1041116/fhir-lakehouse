# FHIR Lakehouse 作品集專案

## 目標

用 Synthea 產生的 FHIR R4 合成資料，建一條 Bronze → Silver → Gold 的 Lakehouse 管線。
程式要能在本機（PySpark + Delta）、Databricks 免費版、Microsoft Fabric 上跑，不綁定平台。

## 環境

- WSL2（Ubuntu）、Java 17、Python 由 uv 管理
- Synthea 資料在 `output/fhir/`（bulk 格式，每種資源一個 NDJSON 檔）
- 產生資料：`java -jar tools/synthea-with-dependencies.jar -p 1000 -s 42 --exporter.fhir.bulk_data true`
- `-p` 是存活病患人數，已故病患也會輸出，所以總人數會多於 `-p`（目前這批：1154 人，其中已故 154 人）

## 目錄

| 路徑 | 用途 |
|---|---|
| `src/ingest/` | Bronze：NDJSON 原樣落地，記錄批次 |
| `src/transform/` | Silver：攤平 FHIR 資源；Gold：星型模型 |
| `src/quality/` | 品質規則與 reject 表 |
| `sql/` | Gold 層的 SQL |
| `tests/` | pytest，用 `tests/fixtures/` 的小量 FHIR 樣本 |
| `docs/` | spec、決策紀錄 |
| `LESSONS.md` | 踩過的坑，每次出錯就加一條 |

## 指令

- 測試：`uv run pytest`
- 檢查：`uv run ruff check .`

## 資料規則

- 任何資料檔、輸出結果都不得 commit
- 攤平時保留 coding 的 system 和 code（SNOMED、LOINC、RxNorm）
- 業務鍵用 FHIR resource id；同一個 id 有多版時，以 `meta.lastUpdated` 取最新，相同時再比批次序號
- 違反品質規則的資料寫入 reject 表，grain 是「規則 × 資料」
- 「增量」要講清楚是抽取增量、計算增量，還是寫入增量

## 寫法

- 轉換優先用 Spark SQL 或 PySpark DataFrame，不用 pandas 處理大量資料
- 平台相關的設定集中在 config，不寫死路徑
- 每次只改指定的檔案或函式，改一小塊就 commit

## 提問方式

- 每個問題標示：現在必須決定／可以先用預設值／之後再處理
- 能給合理預設值的，直接給預設值並說明理由，不要問我
- 同一類錯誤出現第二次，先說明原因（規格沒寫、寫得模糊、工具限制），再修

## 學習模式（我在練 Python）

- 核心轉換函式：先給函式簽名、docstring 和測試，由我實作；我完成後再 review
- 其他部分可以直接實作，但每個檔案完成後，用三到五句說明設計理由

## 完成的定義

- 測試通過，涵蓋缺值、重複 id、多版本、代碼格式錯誤
- 每一層有筆數核對
- 同一批重跑結果一致
- 新踩到的坑已寫進 `LESSONS.md`
