# OpenClaw 記憶體逆向工程彙整摘要

> **Phase 6：Memory Reverse Engineering Summary**
> 文件版本：1.0
> 分析日期：2026-02-08
> 分析範圍：Memory 子系統完整逆向工程彙整

---

## 執行摘要

本文件彙整 OpenClaw Memory 子系統逆向工程的關鍵發現、架構洞察與設計亮點。Memory 是 OpenClaw 中最複雜的子系統之一（43 個檔案、核心 `manager.ts` 達 2,400+ 行），實現了一套「混合式語意搜尋引擎」，讓 AI Agent 能跨會話地記住使用者的工作脈絡。

---

## 1. 架構洞察

### 1.1 核心設計理念

**「讓 AI 記住一切」**：Memory 的設計目標是將使用者的知識筆記與歷史對話轉化為可被 AI 即時檢索的向量索引。這不是簡單的全文搜尋，而是結合了向量語意搜尋（Vector Search）與關鍵字搜尋（FTS5 BM25）的混合架構，確保既能理解語意相似性，也能精確匹配關鍵詞。

### 1.2 三層後備（Triple Fallback）設計

Memory 在每個關鍵層面都實現了後備機制：

| 層面 | 主要策略 | 後備策略 | 最終兜底 |
|------|---------|---------|---------|
| 後端選擇 | QMD 外部引擎 | Builtin 內建引擎 | 回傳空結果 |
| Embedding Provider | 使用者指定 Provider | 自動 Fallback Provider | 拋出可讀錯誤 |
| 向量搜尋 | sqlite-vec 原生計算 | JavaScript 暴力計算 | 純文字搜尋 |
| 全文檢索 | FTS5 + BM25 | 停用（純向量） | 降低搜尋品質 |
| 批次處理 | Batch API | 逐筆 API 呼叫 | 跳過嵌入 |
| 資料庫更新 | 原子性交換 | 保留原始資料庫 | 索引暫不更新 |

這種「漸進式降級」策略確保系統在任何單一元件故障時仍能維持基本功能。

### 1.3 混合搜尋（Hybrid Search）

混合搜尋是 Memory 的核心創新，透過加權融合兩種搜尋結果：

```
最終分數 = vectorWeight × 向量相似度 + textWeight × BM25 分數
```

- **向量搜尋**擅長語意理解（「概念相近」的內容即使用詞不同也能匹配）
- **BM25 搜尋**擅長精確匹配（完全符合查詢關鍵字的內容排名更高）
- 兩者的加權預設各 50%，可由使用者自行調整

### 1.4 Embedding Provider 的外掛式設計

Embedding 層採用典型的 Strategy Pattern（策略模式），所有 Provider 實作統一的 `EmbeddingProvider` 介面（`embedQuery` + `embedBatch`），Manager 僅依賴介面而非具體實作。新增 Provider 只需實作兩個方法即可。

---

## 2. 關鍵發現

### 2.1 檔案結構

- **總檔案數**：43 個 TypeScript 檔案（含 11 個測試檔、15 個以上測試案例）
- **核心複雜度**：`manager.ts`（2,400+ 行）是整個專案中最大的單一模組之一
- **測試覆蓋**：每個核心功能都有專屬測試（原子性重新索引、批次處理、同步錯誤不崩潰等）

### 2.2 資料模型

- **6 張 SQLite 資料表**：meta、files、chunks、chunks_vec、chunks_fts、embedding_cache
- **雙虛擬資料表**：FTS5（全文檢索）+ sqlite-vec（向量索引）
- **分塊策略**：400 tokens / 1,600 chars 為區塊上限，80 tokens / 320 chars 重疊

### 2.3 兩種 Memory 來源

| 來源 | 格式 | 觸發方式 | 特殊處理 |
|------|------|---------|---------|
| `"memory"` | Markdown | 手動撰寫 / Hook / Flush | 標準分塊 |
| `"sessions"` | JSONL | 自動紀錄 | 敏感資訊脫敏、差異追蹤 |

### 2.4 同步機制

四種觸發方式確保索引始終最新：
1. **Watcher**：chokidar 即時偵測檔案變更
2. **事件**：Session Transcript 更新事件
3. **定時**：設定間隔自動同步
4. **搜尋前**：搜尋時發現索引為「髒」則先同步

### 2.5 安全考量

- 符號連結被靜默跳過（防止符號連結攻擊）
- 檔案讀取有路徑安全驗證（防止目錄遍歷）
- Session 紀錄經 `redactSensitiveText()` 脫敏處理
- API 金鑰透過環境變數傳遞，不存於程式碼中

---

## 3. 值得學習的設計

### 3.1 原子性重新索引（Atomic Reindex）

**問題**：重新索引過程中若系統崩潰，可能導致資料庫損毀。

**解法**：在臨時資料庫中完成全部索引工作，成功後透過 `rename` 原子性操作交換主資料庫。失敗時丟棄臨時資料庫，主資料庫完全不受影響。

**位置**：`manager.ts` — `swapIndexFiles()` / `removeIndexFiles()`

### 3.2 Hash 驅動的增量更新

**問題**：每次同步都重新索引所有檔案太浪費。

**解法**：每個檔案計算 SHA-256 雜湊，與資料庫紀錄比對。只有 hash 不同的檔案才會被重新分塊與嵌入。

**位置**：`internal.ts:146-148` — `hashText()`

### 3.3 FallbackMemoryManager 包裝器

**問題**：QMD 外部後端可能在運行中突然失敗。

**解法**：`FallbackMemoryManager`（`search-manager.ts:67-191`）包裝主要 Manager，在首次失敗後自動切換至 builtin MemoryIndexManager，且只需 lazy-load 一次。

### 3.4 Embedding 快取策略

**問題**：Embedding API 呼叫成本高昂。

**解法**：以 `(provider, model, provider_key, hash)` 為主鍵的快取表，相同文字在相同 Provider/Model 下直接命中快取。Provider 變更時快取自動失效。

**位置**：`memory-schema.ts:38-52` — `embedding_cache` 資料表

### 3.5 漸進式增強（Progressive Enhancement）

**問題**：不同部署環境的能力不同（有的有 sqlite-vec、有的沒有）。

**解法**：每個可選元件（sqlite-vec、FTS5、node-llama-cpp）都有獨立的可用性探測和降級路徑。系統在最佳情況下使用所有加速功能，在最差情況下仍能以 JavaScript 純計算運作。

---

## 4. 潛在風險與建議

### 4.1 manager.ts 複雜度過高

- **風險**：2,400+ 行的單一檔案，涵蓋初始化、同步、索引、搜尋、快取、狀態等多個職責
- **建議**：考慮將 Manager 拆分為更小的模組（已部分開始：`sync-memory-files.ts`、`sync-session-files.ts` 是從 Manager 提取出來的）

### 4.2 SQLite 同步 API 的擴展性

- **風險**：`DatabaseSync` 是同步 API，所有資料庫操作阻塞事件循環
- **影響**：大量 chunks 的暴力搜尋（sqlite-vec 不可用時）可能造成延遲
- **建議**：未來可考慮遷移至非同步 SQLite 驅動或 Worker Thread

### 4.3 向量維度鎖定

- **風險**：切換 Embedding Provider 時向量維度可能改變，需要完整重新索引
- **影響**：大型 Memory 庫的重新索引耗時且耗費 API 配額
- **建議**：已有原子性重新索引保護，但建議在設定指引中提醒使用者

### 4.4 Session Memory 的實驗性質

- **風險**：`"sessions"` 來源標記為實驗性功能（`experimental.sessionMemory`）
- **影響**：大量會話紀錄可能顯著增加索引體積和嵌入成本
- **建議**：`deltaBytes` / `deltaMessages` 閾值機制已有效控制更新頻率

---

## 5. 快速上手指南

### 5.1 理解 Memory 的最小閱讀路徑

1. **`types.ts`**（81 行）— 理解核心型別定義，特別是 `MemorySearchManager` 介面
2. **`internal.ts`**（308 行）— 理解分塊演算法與工具函式
3. **`memory-schema.ts`**（97 行）— 理解資料庫結構
4. **`embeddings.ts`**（255 行）— 理解 Provider 選擇與 Fallback 邏輯
5. **`hybrid.ts`**（116 行）— 理解混合搜尋的融合演算法
6. **`search-manager.ts`**（213 行）— 理解工廠模式與 FallbackManager
7. **`manager.ts`**（2,400+ 行）— 最後閱讀核心管理器，先看 `search()` 和 `sync()` 方法

### 5.2 適合修改的切入點

| 難度 | 任務 | 涉及檔案 |
|------|------|---------|
| 簡單 | 新增一個 Embedding Provider | `embeddings-*.ts`（參考現有實作） |
| 簡單 | 調整分塊策略的預設值 | `internal.ts` `chunkMarkdown()` |
| 中等 | 新增搜尋結果排名因子 | `hybrid.ts` `mergeHybridResults()` |
| 中等 | 新增 Memory 來源類型 | `types.ts`、`manager.ts` sync 相關方法 |
| 進階 | 替換 SQLite 為其他儲存後端 | `manager.ts`、`memory-schema.ts`、`manager-search.ts` |
| 進階 | 新增非同步資料庫支援 | `manager.ts` 大量方法需重構 |

---

## 6. 文件交叉參照

| 報告 | 文件名稱 | 主要內容 |
|------|---------|---------|
| Phase 1 | [07_memory_mechanism_discovery.md](07_memory_mechanism_discovery.md) | 檔案清單、概念定義、CRUD 進入點 |
| Phase 2 | [08_memory_architecture_analysis.md](08_memory_architecture_analysis.md) | 分層架構、資料模型、儲存後端、生命週期、索引策略 |
| Phase 3 | [09_memory_data_flow_and_integration.md](09_memory_data_flow_and_integration.md) | 資料流追蹤、跨子系統整合、介面契約、事件驅動 |
| Phase 4 | [10_memory_configuration_and_constraints.md](10_memory_configuration_and_constraints.md) | 設定參數、限制、邊界條件、錯誤處理 |
| Phase 5 | [11_memory_dependency_mapping.md](11_memory_dependency_mapping.md) | 內部依賴、外部依賴、依賴健康度 |
| Phase 6 | [12_memory_reverse_engineering_summary.md](12_memory_reverse_engineering_summary.md) | 本文件 — 彙整摘要 |

---

## 版本紀錄

| 版本 | 日期 | 變更 |
|------|------|------|
| 1.0 | 2026-02-08 | 初版建立 |
