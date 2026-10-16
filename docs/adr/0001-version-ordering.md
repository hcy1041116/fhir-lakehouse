# 0001 版本新舊的判斷順序

- 狀態：accepted（2026-10-16）

## 背景

AGENTS.md 規定：同一個 id 有多個版本時，以 `meta.lastUpdated` 取最新，相同時再比批次序號。

實際檢查 Synthea 的輸出後發現，Patient、Encounter、Condition、Observation 共 683,514 筆，**沒有任何一筆有 `meta.lastUpdated`**，也沒有 `meta.versionId`；`meta` 裡只有 `profile`。照原本的規則，第一步永遠比不出結果。

另外，同一批裡也可能出現同一個 id 兩次，這時批次序號也相同，需要第三個比較條件，結果才會固定。

## 選項

- **A. 補上匯出時間**：有效更新時間 = `meta.lastUpdated`，缺值時用所屬批次的匯出時間；接著比批次序號；最後比檔案中的行號。
- **B. 只比批次序號**：不看 `lastUpdated`。
- **C. 產資料後自己補上 `lastUpdated`**：替每筆資料寫入假的時間。

## 決定

選 A。版本的比較鍵是 **（有效更新時間, 批次序號, 行號）**，三者依序比較，大者為新。

- 匯出時間取 Synthea metadata 檔的 `runStartTime`（實際執行時間）。
- 不用 metadata 的 `endTime`：實驗中某一批的 `endTime` 跳回 `20180614`，不可靠。

## 理由

- 保留 AGENTS.md 的原意，以後接上真正會帶 `lastUpdated` 的 FHIR 來源，不用改規則。
- 在 Synthea 資料上，效果等於「後匯出的批次贏」，符合直覺。
- 每個更新批次都要從上一批的快照產生（見 ADR-0002），批次一定依序產生，所以匯出時間必定隨批次序號遞增，不會互相矛盾。
- B 違反 AGENTS.md，而且遇到真實資料「晚到的舊版本」時會判斷錯。
- C 會讓 Bronze 不再是原樣，違反 Bronze 的定位（ADR-0003）。

## 後果

- 完整的 Synthea 資料測不到「`lastUpdated` 不同」的情況，必須用 `tests/fixtures/` 的手工資料測，包括「晚到的舊版本不能蓋掉新版本」。
- 行號必須能穩定取得，Bronze 要保存每一筆在來源檔中的行號。
