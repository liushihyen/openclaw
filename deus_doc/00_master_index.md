# OpenClaw 逆向工程分析 — 總索引

> **Master Index**
> 文件版本：1.0
> 建立日期：2026-02-08
> 分析對象：OpenClaw — 多通道 AI 代理平台

---

## 專案摘要

OpenClaw 是一個開源的多通道 AI 助理平台，以 TypeScript/Node.js 22+ 為核心，整合 10+ 種通訊平台（WhatsApp、Telegram、Discord、Slack、Signal、iMessage、LINE、Google Chat、Microsoft Teams、Matrix 等），透過統一的 Gateway 閘道與 Envelope 信封系統處理訊息，並以 Agent 層呼叫多種 AI 模型（Anthropic Claude、OpenAI、Google Gemini 等）進行推理回應。程式碼庫規模達 5,000+ 檔案、325,000+ 行 TypeScript，採用 pnpm workspace monorepo 架構，包含 31 個擴展模組與 52 個技能整合。

---

## 文件索引

本系列報告共六份，涵蓋從專案偵察到行動指引的完整逆向工程分析。建議依序號閱讀，或依個人需求跳至特定報告。

### 📋 報告總覽

| 編號 | 文件名稱 | 對應階段 | 頁數（約） | 摘要 |
|------|---------|---------|-----------|------|
| 01 | [專案探索與盤點報告](01_project_inventory.md) | 階段 1：專案總覽偵察 | ~1,170 行 | 建立專案第一印象，涵蓋基本資訊、語言分佈、擴展清單、技能列表、建置與測試策略 |
| 02 | [架構總覽與運作理論](02_architectural_overview.md) | 階段 2–5：架構模式辨識 | ~2,370 行 | 十層架構分析，從 CLI 進入點到記憶體系統的完整分層剖析，含典型訊息處理流程 |
| 03 | [檔案結構地圖](03_file_structure_map.md) | 階段 2：目錄結構解讀 | ~1,120 行 | 完整目錄巡覽，記錄每個目錄與關鍵檔案的功能定位與設定細節 |
| 04 | [依賴與關聯分析](04_dependency_and_relationships.md) | 階段 3：依賴關係拓撲 | ~870 行 | 外部依賴分類、內部模組依賴圖、關鍵呼叫鏈、事件驅動模式、循環依賴識別 |
| 05 | [建置、部署與基礎設施分析](05_build_and_deployment.md) | 階段 9：建置與部署管線 | ~1,430 行 | 建置管線、CI/CD、容器化、雲端部署、品質閘道、原生應用建置流程 |
| 06 | [逆向工程行動指引](06_reverse_engineering_action_items.md) | 階段 10：綜合行動方案 | ~1,270 行 | 閱讀路徑、必讀檔案、複雜度熱區、架構評估、潛在風險、新手切入點 |

---

## 建議閱讀路徑

### 🚀 快速掌握（30 分鐘）

適合需要快速了解專案全貌的人。

1. **本文件（00）** — 掌握六份報告的定位與關聯
2. **[01 專案探索與盤點](01_project_inventory.md)** §執行摘要 + §專案總覽 — 理解 OpenClaw 是什麼、做什麼
3. **[06 行動指引](06_reverse_engineering_action_items.md)** §快速入門路徑 — 得到具體的程式碼閱讀起點

### 🏗️ 架構理解（2 小時）

適合需要理解系統設計的開發者或架構師。

1. **[01 專案盤點](01_project_inventory.md)** — 完整閱讀，建立基礎認知
2. **[02 架構總覽](02_architectural_overview.md)** — 理解十層架構、訊息流、設計哲學
3. **[04 依賴與關聯](04_dependency_and_relationships.md)** — 釐清技術選型與模組互動
4. **[06 行動指引](06_reverse_engineering_action_items.md)** §架構品質評估 + §值得學習的設計 — 了解設計亮點與風險

### 🔧 開發上手（半天）

適合即將開始貢獻程式碼的開發者。

1. **依序閱讀全部六份報告**
2. 特別關注：
   - **[03 檔案結構地圖](03_file_structure_map.md)** — 找到每個模組的確切位置
   - **[05 建置部署](05_build_and_deployment.md)** — 建立本機開發環境
   - **[06 行動指引](06_reverse_engineering_action_items.md)** §新手切入點 — 從簡單任務開始

---

## 各報告核心內容

### 01 — 專案探索與盤點報告

建立對 OpenClaw 的第一印象。涵蓋：

- **基本資訊**：版本（2026.2.6-3）、授權（MIT）、Node.js 22+、ESM 模組系統
- **語言分佈**：TypeScript 佔 325,000+ 行，搭配 Swift（macOS/iOS）、Kotlin（Android）
- **擴展模組**：31 個獨立擴展（1Password、Apple Notes、GitHub、Google 系列等）
- **技能系統**：52 個內建技能，涵蓋日曆、天氣、翻譯、自動化等
- **設定檔全覽**：TypeScript、建置、測試、部署、品質工具的設定檔案清單
- **進入點盤點**：CLI、Gateway、測試等各進入點的啟動指令

### 02 — 架構總覽與運作理論

系統架構的深度剖析。涵蓋：

- **十層架構模型**：CLI Entry → Command → Gateway → Channel Adapter → Agent → Auto-Reply Engine → Infrastructure → Plugin System → Hook System → Memory System
- **架構模式判定**：Multi-Layer SOA + Plugin-Based Extensibility（Monolith 內部）
- **Gateway 層**：HTTP + WebSocket + RPC 三重協定閘道，130+ 個 RPC 方法
- **Channel Adapter**：10+ 通道的統一適配器設計，Envelope 信封系統
- **Agent 層**：多 AI Provider 切換、Streaming 回應、Tool Use 支援
- **記憶體系統**：SQLite + sqlite-vec + FTS5 BM25 + Embedding 混合架構
- **典型訊息流**：從使用者發送到 AI 回應的完整生命週期追蹤

### 03 — 檔案結構地圖

原始碼的導航地圖。涵蓋：

- **根目錄**：60+ 設定檔的功能定位與關鍵設定說明
- **src/ 目錄**：34 個子目錄的逐一剖析（agents、auto-reply、browser、channels、cli、commands、config、gateway、hooks、infra、memory、plugins、providers 等）
- **extensions/ 目錄**：31 個擴展模組的組織方式與命名慣例
- **apps/ 目錄**：原生應用程式（macOS/Swift、iOS、Android/Kotlin）結構
- **scripts/ 目錄**：建置腳本、部署腳本、工具腳本的分類
- **packages/ 目錄**：6 個共用子套件（common、api-client 等）

### 04 — 依賴與關聯分析

技術選型與模組互動。涵蓋：

- **外部依賴分類**：框架層、AI/LLM 層、通道層、資料層、瀏覽器自動化層、媒體處理層、建置工具層、基礎設施層
- **內部依賴圖**：Mermaid 圖表呈現模組間的依賴方向
- **關鍵呼叫鏈**：入站訊息流、Gateway 啟動流程、外掛載入流程的完整追蹤
- **事件模式**：Hook 系統雙層分發、Channel 事件、Session Transcript、Cron、WebSocket RPC
- **循環依賴識別**：auto-reply ↔ agents 的雙向依賴及其影響分析
- **依賴健康度評估**：版本新鮮度、維護狀態、安全考量

### 05 — 建置、部署與基礎設施分析

從原始碼到上線的完整管線。涵蓋：

- **建置管線**：8 步驟建置流程、tsdown 打包設定（4 進入點）
- **TypeScript 設定**：ES2023 目標、NodeNext 模組、嚴格模式
- **測試架構**：7 個 Vitest 設定檔、70% 覆蓋率門檻、矩陣測試
- **容器化**：3 個 Dockerfile（主服務、沙箱、瀏覽器沙箱）、Docker Compose
- **雲端部署**：Fly.io（公開/私有）、Render
- **CI/CD**：GitHub Actions 4 個 Workflow（ci、docker-release、formal-conformance、install-smoke）
- **品質閘道**：Pre-commit hooks 9+ 工具鏈（Oxlint、Oxfmt、typos、secrets 偵測等）
- **原生應用建置**：Swift/SwiftUI（macOS/iOS）、Kotlin/Gradle（Android）、Fastlane CI

### 06 — 逆向工程行動指引

可執行的學習與開發行動方案。涵蓋：

- **閱讀路徑**：快速入門（30 分鐘）、核心架構（2 小時）、擴展系統（1 小時）、通道實作（1.5 小時）
- **必讀檔案清單**：依優先級分類的關鍵原始碼檔案
- **複雜度熱區**：5 個高複雜度區域（Gateway 方法分發、Auto-Reply 決策引擎、Channel 適配器矩陣、Plugin 生命週期、Memory 混合架構）
- **架構品質評估**：7 個面向的 1–5 分評分
- **值得學習的設計**：5 個值得借鏡的設計模式
- **潛在風險**：7 個風險項目與改善建議
- **新手切入點**：簡單、中等、進階三級難度的貢獻任務建議
- **本機開發環境建置**：step-by-step 的環境設定指引

---

## 術語對照

本系列報告遵循台灣繁體中文用語規範，以下為常用術語對照：

| 本報告使用 | 其他常見用法 | 英文原文 |
|-----------|------------|---------|
| 使用者 | 用戶 | User |
| 回饋 | 反饋 | Feedback |
| 資料 | 數據 | Data |
| 設定 | 配置 | Configuration |
| 元件 | 組件 | Component |
| 支援 | 支持 | Support |
| 整合 | 集成 | Integration |
| 實作 | 實施 | Implementation |
| 樣版 | 模板 | Template |
| 擴展 | 擴展/擴充 | Extension |
| 外掛 | 插件 | Plugin |
| 閘道 | 網關 | Gateway |
| 頻道 / 通道 | 渠道 | Channel |

---

## 分析方法論

本系列報告依照 `CLAUDE.md` 定義的十階段分析框架執行：

```
階段 1 ─ 專案總覽偵察        → 01_project_inventory.md
階段 2 ─ 目錄結構解讀        → 03_file_structure_map.md
階段 3 ─ 依賴關係拓撲        → 04_dependency_and_relationships.md
階段 4 ─ 進入點與啟動流程    → 02_architectural_overview.md（含啟動序列分析）
階段 5 ─ 核心架構模式辨識    → 02_architectural_overview.md（含架構模式判定）
階段 6 ─ 資料流與狀態管理    → 02_architectural_overview.md（含典型訊息流追蹤）
階段 7 ─ API 介面與對外契約  → 02_architectural_overview.md（含 Gateway RPC 方法）
階段 8 ─ 測試策略與品質保障  → 05_build_and_deployment.md（含測試架構分析）
階段 9 ─ 建置與部署管線      → 05_build_and_deployment.md
階段 10 ─ 綜合分析報告產出   → 06_reverse_engineering_action_items.md
```

### 分析原則

1. **先觀察，再推論**：所有結論基於程式碼中的具體證據，標明檔案路徑
2. **區分事實與推測**：確定的事實直接陳述，推測部分使用「推測：」前綴
3. **關注設計意圖**：嘗試理解「為什麼這樣設計」而非只描述「設計了什麼」
4. **追蹤邊界與例外**：重視錯誤處理、邊界條件等容易被忽略的面向
5. **保持實用導向**：所有輸出服務於「讀完後可以開始貢獻程式碼」的目標
6. **承認不確定性**：無法確認的細節明確標示，避免猜測作答

---

## 版本紀錄

| 版本 | 日期 | 變更內容 |
|------|------|---------|
| 1.0 | 2026-02-08 | 初版建立，包含全部六份報告的索引與摘要 |

---

> 「理解一個專案的最好方式，是跟著資料走一遍它的旅程。從進入系統的那一刻開始，經過每一層的處理，到最後回傳給使用者。走完這趟旅程，你就真的懂了。」
