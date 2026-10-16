# 0007 不綁定平台：邏輯表名加上各環境的 config

- 狀態：accepted（2026-10-16）

## 背景

程式要能在本機（PySpark + Delta）和 Databricks 免費版上跑，之後還要能支援 Microsoft Fabric。兩邊的條件不同：

| | 本機 | Databricks 免費版 |
|---|---|---|
| 運算 | PySpark 4.2.0 + delta-spark 4.4.1（實測可寫入和 MERGE） | 只有 serverless |
| 表的位置 | 沒有 metastore，用路徑存取最簡單 | Unity Catalog，用 `catalog.schema.table` 存取 |
| 來源檔 | 本機資料夾 | Unity Catalog Volume |
| API 限制 | 無 | 不支援 RDD API |
| Python | 3.12.3（uv 管理） | 3.12.3（serverless environment 4、5） |
| Spark 預設時區 | `Asia/Taipei` | UTC |

## 選項

- **A. 全部用路徑**：本機可以，但 Unity Catalog 不建議、也不方便用路徑管理表。
- **B. 全部用表名**：Databricks 可以，但本機要另外架 Hive metastore，否則表名在 Spark 重開後就消失了。
- **C. 邏輯表名加上解析層**：程式只寫邏輯表名（例如 `silver.patient`），由各環境的 config 決定實際位置。

## 決定

選 C。

- 程式裡只出現邏輯表名，格式是「層.表」：`bronze`、`silver`、`gold`、`quality`、`ops`。
- 各環境一份 TOML config，內容包括：
  - 表的存放方式：本機是 lakehouse 根目錄底下的 Delta 路徑；Databricks 是 catalog 名稱，對應到 Unity Catalog 的受管理表
  - 來源批次資料的根目錄
  - Spark 時區（固定 UTC，見 ADR-0008）
- 本機由入口程式自己建立帶 Delta 的 Spark session；Databricks 直接使用平台提供的 session。
- 程式碼的限制：不使用 RDD 和 SparkContext。
- 本機的 Python 也固定為 3.12，和 Databricks serverless 相同。用到新版才有的語法時，本機就會先出錯，不用等到上 Databricks 才發現。
- Fabric 第一階段不做，之後在解析層加一種模式即可。

## 理由

- 平台差異集中在一個地方，轉換邏輯完全不用知道自己在哪裡跑，符合 AGENTS.md「平台相關的設定集中在 config」。
- 本機不用額外架 metastore，Databricks 又能用最自然的 Unity Catalog 表名。
- TOML 是 Python 內建就能讀的格式，不用多裝套件。

## 後果

- 每次讀寫表都要經過解析層，不能直接呼叫 `spark.table()` 或寫死路徑。
- `pyproject.toml` 原本要求 Python `>=3.13`，已改為 `>=3.12`，`.python-version` 改為 `3.12`。改完後本機實測 Spark + Delta 的寫入和 MERGE 正常。
- 第一階段的 Databricks 驗收，以「用同一份程式跑通一批」為準。
