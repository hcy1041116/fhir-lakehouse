# FHIR Lakehouse

把 Synthea 產生的 FHIR R4 合成病歷，經過 Bronze → Silver → Gold 整理成可分析的資料。這份名詞表定義專案裡每個詞的意思，文件、程式命名、票都用這裡的說法。

## 批次與資料來源

**批次（Batch）**：
一次 Synthea 執行的完整輸出，四種資源一起。批次是載入、核對、重跑的最小單位。
_Avoid_：load、run、匯入

**批次序號（Batch sequence）**：
批次的整數順序，從 1 開始，代表批次產生的先後。
_Avoid_：批次編號（容易和 batch_id 混淆）

**初始批次（Initial batch）**：
資料截止日最早的批次，也就是第一批。

**更新批次（Update batch）**：
同一群病人模擬到更晚的資料截止日所產生的批次，用來模擬來源系統的定期更新。
_Avoid_：快照（本專案不使用 Synthea 的快照功能）

**資料截止日（As-of date）**：
一個批次代表的模擬日期，也就是「這批資料是來源系統在哪一天的樣子」。這是批次層級的時間，同一批的每筆資料都一樣，不是每筆資料各自的更新時間。
_Avoid_：匯出時間、執行時間（實際產生資料的時刻和資料代表的日期無關）

## 版本

**版本（Version）**：
同一個 FHIR resource id 在某一批中出現的一筆資料。同一個 id 可以有很多版本。

**有效更新時間（Effective last-updated）**：
用來比較版本新舊的時間。資料本身有 `meta.lastUpdated` 就用它，沒有就用所屬批次的資料截止日。

**最新版本（Current version）**：
同一個 id 所有版本中，有效更新時間最晚的那一版；相同時取批次序號大的，再相同時取檔案中較後面的一行。
_Avoid_：最後載入的版本（晚載入的不一定比較新）

**未出現（Absent）**：
某個 id 在之前的批次有、這一批沒有。只代表它不在這次的匯出範圍內，**不是刪除**。
_Avoid_：消失、刪除

**刪除（Deletion）**：
來源明確表示某筆資料作廢或刪除。第一階段不處理。

## 品質

**品質規則（Quality rule）**：
對單筆資料的一項檢查，有固定代號和嚴重度。

**嚴重度（Severity）**：
品質規則的等級。`error`：資料被擋下；`warn`：資料照常使用，但留下 reject 紀錄。

**擋下（Blocked）**：
資料因為違反 `error` 級規則而不進入 Silver。

**Reject 紀錄（Reject record）**：
「一筆資料違反一條規則」的紀錄。一筆資料違反三條規則就有三筆。Reject 紀錄不一定代表資料被擋下，要看嚴重度。
_Avoid_：錯誤資料、髒資料

## 臨床概念

**主要代碼（Primary coding）**：
一筆資源有多個 coding 時，代表它的那一個：優先選該資源預期的代碼系統（Condition 用 SNOMED，Observation 用 LOINC）。

**診斷（Diagnosis）**：
SNOMED 語意標籤為 `disorder` 的 Condition。`finding`（例如壓力）、`situation`（例如該做用藥檢查）等不算診斷。
_Avoid_：把所有 Condition 都叫診斷

**發病年齡（Age at onset）**：
病人在該 Condition 發生當下的年齡。

**年齡層（Age band）**：
分析用的年齡分組：0–17、18–44、45–64、65+。

**記錄日期（As-recorded date）**：
時間字串上原本寫的日曆日期，不換算時區。

## 增量與一致性

**抽取增量（Incremental extraction）**：
只從來源取出有變動的資料。本專案不做：Synthea 每批都是完整匯出。

**計算增量（Incremental computation）**：
每次只處理新批次的資料，和已整理好的資料比對。

**寫入增量（Incremental write）**：
只寫入新增或內容有變的資料，沒變的不重寫。

**重跑一致（Rerun consistency）**：
同一批資料重跑之後，除了載入時間這類稽核欄位，所有表的內容都和上次完全相同。
