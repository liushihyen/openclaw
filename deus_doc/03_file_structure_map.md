# OpenClaw 專案檔案結構地圖

> **階段 3：原始碼檔案結構地圖（Source Code File Structure Map）**
>
> 本文件詳細記錄 OpenClaw 專案的完整目錄結構、檔案配置與功能定位，
> 作為理解專案架構的第一手參考資料。

---

## 文件概要

- **專案名稱**：OpenClaw
- **專案類型**：AI 驅動的多通道對話系統（Monorepo 架構）
- **總檔案數量**：約 5,000+ 個檔案
- **主要程式語言**：TypeScript、Swift、Kotlin、Go
- **建置工具**：tsdown、pnpm、Vite、Gradle、Swift Package Manager
- **測試框架**：Vitest（單元測試、E2E 測試、閘道伺服器測試）
- **容器化**：Docker、Docker Compose
- **部署平台**：Fly.io、Render

---

## 根目錄檔案清單

根目錄包含專案的核心設定檔、建置腳本、部署設定與文件入口。

| 檔案名稱 | 功能定位 | 關鍵設定 |
|---------|---------|---------|
| `package.json` | 專案主要設定檔 | 60+ 依賴套件、80+ 建置腳本、pnpm workspace、exports 匯出點 |
| `openclaw.mjs` | 分散式二進位進入點 | 啟用編譯快取、匯入 `dist/entry.js` |
| `tsconfig.json` | TypeScript 主要設定 | ES2023、NodeNext 模組、嚴格模式、experimentalDecorators |
| `tsconfig.plugin-sdk.dts.json` | 外掛 SDK 型別產生設定 | 用於產生外掛 API 型別定義檔 |
| `tsdown.config.ts` | tsdown 建置工具設定 | 4 個建置進入點：index、entry、plugin-sdk、extensionAPI |
| `vitest.config.ts` | Vitest 主要測試設定 | 120 秒逾時、fork pool、70% 覆蓋率門檻 |
| `vitest.unit.config.ts` | 單元測試設定 | 排除 gateway、extensions 目錄 |
| `vitest.e2e.config.ts` | E2E 測試設定 | 端對端整合測試 |
| `vitest.gateway.config.ts` | 閘道伺服器測試設定 | 專門測試 Gateway Server |
| `vitest.extensions.config.ts` | 擴充套件測試設定 | 測試所有擴充套件模組 |
| `vitest.live.config.ts` | 即時測試設定 | 單一行程執行模式 |
| `Dockerfile` | 主要應用程式容器 | 基於 node:22-bookworm、非 root 使用者 |
| `Dockerfile.sandbox` | AI 代理沙盒容器 | 基於 debian:bookworm-slim |
| `Dockerfile.sandbox-browser` | 瀏覽器沙盒容器 | 包含 Chromium + VNC + noVNC |
| `docker-compose.yml` | Docker Compose 服務定義 | Gateway + CLI 服務 |
| `fly.toml` | Fly.io 公開部署設定 | public、shared-cpu-2x、2GB RAM |
| `fly.private.toml` | Fly.io 私有部署設定 | 無公開 ingress |
| `render.yaml` | Render 平台部署設定 | 部署至 Render.com |
| `.oxlintrc.json` | Oxlint 規則設定 | 程式碼品質檢查規則 |
| `.oxfmtrc.jsonc` | Oxfmt 格式化設定 | 程式碼格式化規則 |
| `.pre-commit-config.yaml` | Pre-commit 鉤子鏈設定 | Git commit 前的自動檢查 |
| `AGENTS.md` | AI 編碼代理指南 | 專案開發規範與建議 |
| `CHANGELOG.md` | 版本歷史記錄 | 從 2026.2.6 開始的變更日誌 |
| `README.md` | 專案總覽文件 | 專案簡介、安裝、使用說明 |
| `LICENSE` | MIT 授權條款 | 開源授權聲明 |
| `appcast.xml` | macOS Sparkle 自動更新源 | 應用程式自動更新資訊 |
| `.env.example` | 環境變數範本 | Twilio 憑證等環境變數範例 |

---

## src/ 目錄（2,606 個檔案 — 核心原始碼）

### src/agents/（150+ 個檔案）

**功能定位**：AI 代理系統核心，負責工作區管理、身份識別、模型設定、工具政策、技能管理、子代理與沙盒執行。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `workspace-management/` | 工作區管理：專案脈絡追蹤、檔案系統操作、權限控制 |
| `identity/` | 代理身份系統：使用者識別、權限驗證、角色管理 |
| `models-config/` | 模型設定管理：AI 模型選擇、參數調整、提供者切換 |
| `tool-policy/` | 工具使用政策：工具權限控制、使用限制、安全規則 |
| `skills/` | 技能系統：技能註冊、載入、執行、依賴管理 |
| `subagents/` | 子代理管理：代理委派、協作、結果彙整 |
| `sandbox-execution/` | 沙盒執行環境：隔離執行、資源限制、安全容器 |

**設計特點**：
- 採用模組化設計，各子系統可獨立運作
- 工具政策與技能系統分離，提升安全性
- 沙盒執行確保程式碼安全執行

---

### src/auto-reply/（200+ 個檔案）

**功能定位**：核心訊息處理引擎，負責訊息封裝、指令偵測與分派、代理執行、回覆管線與提供者調度。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `envelope-system/` | 訊息封裝系統：標準化訊息格式、元資料管理、序列化 |
| `command-detection/` | 指令偵測：解析使用者輸入、識別指令意圖 |
| `command-dispatch/` | 指令分派：路由至對應處理器、優先權管理 |
| `agent-runner/` | 代理執行器：啟動代理、監控執行、錯誤處理 |
| `reply-pipeline/` | 回覆管線：訊息組裝、格式轉換、傳送排程 |
| `provider-dispatcher/` | 提供者調度：AI 提供者選擇、負載平衡、容錯切換 |
| `templating/` | 樣版系統：訊息樣版、參數替換、多語言支援 |

**設計特點**：
- Envelope 模式統一訊息格式，降低通道差異
- Pipeline 架構支援訊息處理的擴充與客製化
- Provider Dispatcher 實作多提供者容錯機制

---

### src/browser/（93 個檔案）

**功能定位**：基於 Playwright 的瀏覽器自動化系統，整合 Chrome DevTools Protocol、頁面脈絡管理、ARIA 快照與 HTTP 路由。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `cdp-integration/` | Chrome DevTools Protocol 整合：底層控制、效能分析 |
| `page-context/` | 頁面脈絡管理：DOM 追蹤、狀態同步、快照管理 |
| `aria-snapshots/` | ARIA 樹狀快照：無障礙結構擷取、語義分析 |
| `chrome-detection/` | Chrome 瀏覽器偵測：版本檢查、路徑自動偵測 |
| `profile-management/` | 使用者設定檔管理：Cookies、LocalStorage、Session |
| `http-routes/` | HTTP 路由代理：請求攔截、回應修改、快取管理 |

**設計特點**：
- CDP 整合提供精細的瀏覽器控制能力
- ARIA 快照支援無障礙介面理解
- Profile 管理實現多使用者情境隔離

---

### src/canvas-host/（3 個檔案）

**功能定位**：原生應用程式的 Canvas 託管服務，提供靜態檔案伺服、WebSocket 即時重載與 A2UI 協定橋接。

| 檔案 | 功能說明 |
|------|---------|
| `static-file-server.ts` | 靜態檔案伺服：HTML、CSS、JS 資源託管 |
| `websocket-live-reload.ts` | WebSocket 即時重載：開發時自動重新整理 |
| `a2ui-protocol-bridge.ts` | A2UI 協定橋接：原生 UI 與 Web UI 通訊 |

**設計特點**：
- 輕量級設計，僅 3 個檔案實作完整 Canvas 託管
- WebSocket 即時重載提升開發體驗
- A2UI 協定橋接實現原生與 Web UI 無縫整合

---

### src/channels/（33 個項目）

**功能定位**：通道註冊與管理，包含 Dock 系統（能力定義、執行緒管理、提及偵測）、白名單、外掛正規化與輸出、傳送者身份與指令閘門。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `registry/` | 通道註冊表：所有通道的中央註冊與查詢 |
| `dock-system/` | Dock 系統：能力定義、執行緒管理、提及偵測 |
| `capabilities/` | 能力定義：各通道支援的功能集合 |
| `threading/` | 執行緒管理：對話串追蹤、回覆關聯 |
| `mentions/` | 提及偵測：@使用者、@機器人偵測與處理 |
| `allowlists/` | 白名單管理：通道存取控制、使用者授權 |
| `plugin-normalize/` | 外掛正規化：統一外掛訊息格式 |
| `plugin-outbound/` | 外掛輸出：訊息傳送至外掛通道 |
| `plugin-actions/` | 外掛動作：外掛指令處理 |
| `sender-identity/` | 傳送者身份：使用者識別、匿名處理 |
| `command-gating/` | 指令閘門：指令權限控制、速率限制 |

**設計特點**：
- 通道抽象層統一多平台差異
- Dock 系統提供可擴充的能力定義機制
- 白名單與閘門確保安全性

---

### src/cli/（110+ 個檔案）

**功能定位**：基於 Commander.js 的 CLI 程式建構器，包含指令註冊、延遲載入子指令、外掛 CLI 指令、提示輔助工具、依賴管理與設定檔管理。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `program-builder/` | CLI 程式建構：Commander.js 設定、全域選項 |
| `command-registry/` | 指令註冊表：動態註冊、指令查詢 |
| `lazy-loading/` | 延遲載入：按需載入子指令，加速啟動 |
| `plugin-cli-commands/` | 外掛 CLI 指令：外掛提供的 CLI 擴充 |
| `prompt-helpers/` | 提示輔助工具：互動式輸入、確認對話框 |
| `deps-management/` | 依賴管理：套件安裝、版本檢查 |
| `profile-management/` | 設定檔管理：使用者設定檔建立、切換、刪除 |

**設計特點**：
- 延遲載入策略加速 CLI 啟動時間
- 外掛系統支援第三方指令擴充
- 豐富的提示輔助工具提升使用體驗

---

### src/commands/（178 個檔案）

**功能定位**：CLI 指令處理器集合，涵蓋專案初始化、代理管理、通道操作、訊息處理、模型設定、健康檢查、記憶體管理、技能管理等。

| 指令類別 | 指令範例 | 功能說明 |
|---------|---------|---------|
| 初始化 | `onboard` | 專案初始化、設定嚮導 |
| 代理管理 | `agent` | 代理建立、啟動、停止、狀態查詢 |
| 通道操作 | `channels` | 通道列表、新增、移除、設定 |
| 訊息處理 | `messages` | 訊息傳送、查詢、刪除 |
| 模型設定 | `models` | AI 模型列表、切換、設定 |
| 健康檢查 | `health`、`doctor` | 系統健康檢查、問題診斷與修復 |
| 狀態查詢 | `status` | 服務狀態、執行時資訊 |
| 記憶體管理 | `memory` | 記憶體查詢、清除、匯出 |
| 設定管理 | `config` | 設定檔讀寫、驗證 |
| 技能管理 | `skills` | 技能列表、啟用、停用 |
| 節點管理 | `nodes` | 分散式節點管理 |
| 排程任務 | `cron` | Cron 任務建立、管理 |
| 背景服務 | `daemon` | 背景服務啟動、停止 |
| 瀏覽器控制 | `browser` | 瀏覽器自動化控制 |
| 互動嚮導 | `wizard` | 互動式設定嚮導 |

**設計特點**：
- 每個指令獨立檔案，職責單一
- 豐富的指令集涵蓋所有管理需求
- `doctor` 指令提供自動診斷與修復建議

---

### src/config/（125+ 個檔案）

**功能定位**：設定管理系統，支援 JSON5 I/O、自動備份輪替、環境變數替換、Include 指令（含循環偵測）、Zod 結構驗證與 Session 管理（45 秒 TTL 快取）。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `json5-io/` | JSON5 讀寫：支援註解、尾隨逗號的 JSON5 格式 |
| `backup-rotation/` | 備份輪替：自動建立備份、保留 N 個歷史版本 |
| `env-substitution/` | 環境變數替換：`${ENV_VAR}` 語法、預設值支援 |
| `includes/` | Include 指令：設定檔模組化、循環參考偵測 |
| `zod-validation/` | Zod 驗證：結構驗證、型別安全、錯誤訊息 |
| `sessions/` | Session 管理：Session 儲存、45 秒 TTL 快取 |
| `store/` | 設定儲存：檔案系統儲存、原子寫入 |
| `transcript/` | 對話記錄：對話歷史、元資料追蹤 |
| `metadata/` | 元資料管理：設定檔元資料、版本追蹤 |
| `paths/` | 路徑解析：設定檔路徑、工作目錄 |

**設計特點**：
- JSON5 支援提升設定檔可讀性
- 備份輪替機制防止設定遺失
- Include 循環偵測避免無窮遞迴
- Zod 驗證確保設定檔結構正確性
- 45 秒 TTL 快取平衡效能與即時性

---

### src/cron/（48 個檔案）

**功能定位**：持久化 Cron 任務系統，包含服務狀態管理、CRUD 操作、檔案系統儲存、計時器管理與隔離代理執行。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `service-state/` | 服務狀態：Cron 服務啟動、停止、重啟狀態 |
| `crud-operations/` | CRUD 操作：任務建立、讀取、更新、刪除 |
| `file-system-store/` | 檔案系統儲存：持久化任務定義至檔案 |
| `timer-management/` | 計時器管理：Cron 表達式解析、排程執行 |
| `isolated-agent-execution/` | 隔離代理執行：每個任務獨立代理、資源隔離 |
| `delivery-to-channels/` | 通道傳送：任務結果傳送至指定通道 |

**設計特點**：
- 檔案系統儲存確保任務持久化
- 隔離執行避免任務互相干擾
- 支援將結果傳送至任意通道

---

### src/daemon/（31 個檔案）

**功能定位**：跨平台背景服務管理，支援 macOS（launchd plist）、Linux（systemd unit）、Windows（Task Scheduler），統一 GatewayService 介面。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `gateway-service/` | 統一閘道服務介面：跨平台抽象層 |
| `macos-launchd/` | macOS launchd 整合：plist 產生、服務管理 |
| `linux-systemd/` | Linux systemd 整合：unit 檔案產生、服務管理 |
| `windows-task-scheduler/` | Windows 工作排程器整合：XML 產生、服務管理 |

**設計特點**：
- 統一介面抽象平台差異
- 自動產生平台特定的設定檔
- 支援服務安裝、解除安裝、啟動、停止

---

### src/discord/（80+ 個檔案）

**功能定位**：Discord 通道整合，基於 @buape/carbon WebSocket 閘道，包含訊息處理（預檢 + 處理）、執行緒管理、執行批准、原生指令與訊息傳送（輸出、反應、表情符號）。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `websocket-gateway/` | @buape/carbon WebSocket 閘道整合 |
| `message-handler/` | 訊息處理器：預檢驗證 + 訊息處理 |
| `preflight/` | 預檢驗證：權限檢查、速率限制 |
| `process/` | 訊息處理：解析、路由、回覆 |
| `threading/` | 執行緒管理：對話串建立、回覆關聯 |
| `exec-approvals/` | 執行批准：敏感操作需使用者確認 |
| `native-commands/` | 原生指令：Discord Slash Commands |
| `send/` | 訊息傳送系統 |
| `outbound/` | 輸出訊息：文字、嵌入式、檔案 |
| `reactions/` | 反應管理：新增、移除表情符號反應 |
| `emojis/` | 表情符號：自訂表情符號處理 |

**設計特點**：
- WebSocket 閘道提供即時訊息接收
- 預檢機制確保安全性
- 執行緒管理支援複雜對話結構
- Slash Commands 提供原生 Discord 體驗

---

### src/gateway/（130+ 個檔案）

**功能定位**：閘道伺服器核心，整合 HTTP/WebSocket 伺服器、認證（JWT/API Key/loopback）、RPC 方法分派（約 50+ 個方法）、Session 管理、通道生命週期、瀏覽器控制、Canvas UI 託管與 Tailscale 暴露。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `http-server/` | HTTP 伺服器：Express 基礎、路由設定 |
| `websocket-server/` | WebSocket 伺服器：即時雙向通訊 |
| `auth/` | 認證系統 |
| `jwt/` | JWT 認證：Token 簽發、驗證 |
| `api-key/` | API Key 認證：金鑰管理、驗證 |
| `loopback/` | Loopback 認證：本機免認證存取 |
| `rpc-dispatch/` | RPC 方法分派：約 50+ 個 RPC 方法 |
| `session-management/` | Session 管理：Session 建立、追蹤、清理 |
| `channel-lifecycle/` | 通道生命週期：通道啟動、停止、重啟 |
| `browser-control/` | 瀏覽器控制：遠端瀏覽器操作 API |
| `canvas-ui-hosting/` | Canvas UI 託管：Web UI 伺服 |
| `tailscale-exposure/` | Tailscale 暴露：私有網路存取 |

**設計特點**：
- HTTP + WebSocket 雙協定支援
- 多重認證機制適應不同情境
- 50+ RPC 方法提供完整管理能力
- Tailscale 整合實現安全遠端存取

---

### src/hooks/（30 個檔案）

**功能定位**：事件驅動鉤子系統，包含內部鉤子（全域 Map 註冊表）、載入器（動態匯入 + 快取破壞）、4 種來源（bundled、managed、workspace、plugin）與內建鉤子（boot-md、command-logger、session-memory）。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `internal-hooks/` | 內部鉤子：全域 Map 註冊表、事件發布訂閱 |
| `loader/` | 鉤子載入器：動態匯入、快取破壞 |
| `sources/` | 鉤子來源管理 |
| `bundled/` | 內建鉤子：隨專案打包的鉤子 |
| `managed/` | 管理鉤子：使用者安裝的鉤子 |
| `workspace/` | 工作區鉤子：專案特定鉤子 |
| `plugin/` | 外掛鉤子：外掛提供的鉤子 |
| `bundled-hooks/` | 內建鉤子實作 |
| `boot-md/` | 啟動 Markdown 鉤子：啟動時執行 Markdown 腳本 |
| `command-logger/` | 指令記錄器：記錄所有 CLI 指令 |
| `session-memory/` | Session 記憶鉤子：Session 狀態持久化 |

**設計特點**：
- 四層來源機制提供靈活的鉤子管理
- 動態載入支援熱更新
- 內建鉤子涵蓋常見使用情境

---

### src/imessage/（17 個檔案）

**功能定位**：iMessage 通道整合，透過 BlueBubbles RPC 客戶端實現監控提供者、目標解析（handle、service）與傳送管理。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `bluebubbles-rpc/` | BlueBubbles RPC 客戶端：與 BlueBubbles 伺服器通訊 |
| `monitor-provider/` | 監控提供者：監聽新訊息 |
| `target-resolution/` | 目標解析：handle（電話 / Email）、service（iMessage / SMS） |
| `delivery-management/` | 傳送管理：訊息傳送、重試、狀態追蹤 |

**設計特點**：
- BlueBubbles RPC 實現 macOS iMessage 整合
- 支援 iMessage 與 SMS 雙模式
- 完整的傳送管理與錯誤處理

---

### src/infra/（153 個檔案）

**功能定位**：基礎設施模組，包含環境管理（shell-env fallback）、OS 摘要、裝置認證、SSH Tunnel、SSRF 防禦、Fetch Guard、心跳執行器、系統事件、Bonjour 探索、提供者使用 / 成本追蹤與埠號管理。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `env-management/` | 環境管理：環境變數載入、shell-env fallback |
| `os-summary/` | OS 摘要：作業系統資訊收集 |
| `device-auth/` | 裝置認證：裝置綁定、授權 |
| `ssh-tunnel/` | SSH Tunnel：安全通道建立 |
| `ssrf-defense/` | SSRF 防禦：伺服器端請求偽造防護 |
| `fetch-guard/` | Fetch Guard：HTTP 請求安全包裝 |
| `heartbeat-runner/` | 心跳執行器：服務健康檢查 |
| `system-events/` | 系統事件：系統級事件監聽 |
| `bonjour-discovery/` | Bonjour 探索：區域網路服務探索 |
| `provider-usage-tracking/` | 提供者使用追蹤：API 呼叫統計 |
| `provider-cost-tracking/` | 提供者成本追蹤：API 成本計算 |
| `port-management/` | 埠號管理：埠號分配、衝突檢測 |

**設計特點**：
- 完整的基礎設施抽象層
- SSRF 防禦與 Fetch Guard 確保安全性
- 提供者成本追蹤幫助控制 AI API 支出
- Bonjour 探索支援零設定區域網路通訊

---

### src/line/（38 個檔案）

**功能定位**：LINE 通道整合，基於 @line/bot-sdk，包含 Webhook 簽章驗證、Flex Messages、Rich Menu、Template Messages、Markdown 轉 LINE 格式與回覆分塊。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `bot-sdk/` | @line/bot-sdk 整合 |
| `webhook-verification/` | Webhook 簽章驗證：確保訊息來源 |
| `flex-messages/` | Flex Messages：豐富的卡片式訊息 |
| `rich-menu/` | Rich Menu：自訂選單介面 |
| `template-messages/` | Template Messages：按鈕、確認、輪播樣版 |
| `markdown-to-line/` | Markdown 轉 LINE 格式：格式轉換 |
| `reply-chunks/` | 回覆分塊：長訊息分段傳送 |

**設計特點**：
- 完整支援 LINE 豐富訊息格式
- Markdown 轉換提升內容相容性
- 回覆分塊解決 LINE 訊息長度限制

---

### src/link-understanding/（7 個檔案）

**功能定位**：連結偵測與理解，包含 URL 擷取、頁面摘要與訊息附件處理。

| 檔案 | 功能說明 |
|------|---------|
| `url-extraction.ts` | URL 擷取：從文字中提取 URL |
| `page-summary.ts` | 頁面摘要：爬取網頁並產生摘要 |
| `message-attachment.ts` | 訊息附件：將摘要附加至訊息 |

**設計特點**：
- 自動偵測訊息中的 URL
- AI 驅動的頁面內容摘要
- 無縫整合至訊息流

---

### src/logging/（17 個檔案）

**功能定位**：結構化日誌系統，包含子系統記錄器、檔案日誌等級與主控台擷取。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `subsystem-logger/` | 子系統記錄器：各模組獨立記錄器 |
| `file-log-levels/` | 檔案日誌等級：不同檔案不同等級 |
| `console-capture/` | 主控台擷取：擷取 stdout/stderr |

**設計特點**：
- 結構化日誌支援查詢與分析
- 子系統記錄器實現模組化日誌管理
- 主控台擷取確保所有輸出都被記錄

---

### src/media/（20 個檔案）

**功能定位**：媒體管理，包含媒體託管（Tailscale、TTL）、Express 伺服器、檔案儲存、MIME 偵測、音訊 / 圖片操作與媒體下載。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `hosting/` | 媒體託管：Tailscale 私有託管、TTL 過期 |
| `express-server/` | Express 伺服器：靜態檔案伺服 |
| `file-store/` | 檔案儲存：媒體檔案管理 |
| `mime-detection/` | MIME 偵測：檔案類型自動識別 |
| `audio-operations/` | 音訊操作：轉碼、剪輯、合併 |
| `image-operations/` | 圖片操作：縮放、裁切、格式轉換 |
| `fetch/` | 媒體下載：URL 媒體下載 |

**設計特點**：
- Tailscale 私有託管確保安全性
- TTL 機制自動清理過期媒體
- 完整的音訊與圖片處理能力

---

### src/media-understanding/（41 個檔案）

**功能定位**：多提供者媒體 AI 分析，整合 Anthropic（視覺）、OpenAI（Whisper/GPT-4V）、Google（VideoIntelligence）、Deepgram（即時音訊）、Groq、Minimax，包含並行控制與容錯切換。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `anthropic-vision/` | Anthropic 視覺：圖片理解 |
| `openai-whisper/` | OpenAI Whisper：語音轉文字 |
| `openai-gpt4v/` | OpenAI GPT-4V：圖片理解 |
| `google-video-intelligence/` | Google VideoIntelligence：影片分析 |
| `deepgram-live-audio/` | Deepgram 即時音訊：即時語音轉文字 |
| `groq/` | Groq：快速 AI 推論 |
| `minimax/` | Minimax：多模態 AI |
| `concurrency-control/` | 並行控制：請求限流、排程 |
| `fallback/` | 容錯切換：提供者失敗時自動切換 |

**設計特點**：
- 多提供者支援確保可用性
- 並行控制避免 API 配額超限
- 容錯切換機制確保服務穩定性

---

### src/memory/（分散於核心與擴充套件）

**功能定位**：混合搜尋系統，整合 SQLite + sqlite-vec 向量、FTS5 BM25、嵌入提供者（OpenAI/Gemini/Voyage）、批次最佳化與 Session 同步。

| 元件 | 功能說明 |
|------|---------|
| SQLite + sqlite-vec | 向量儲存：高效能向量相似度搜尋 |
| FTS5 BM25 | 全文搜尋：傳統關鍵字搜尋 |
| 嵌入提供者 | OpenAI/Gemini/Voyage：文字向量化 |
| 批次最佳化 | 批次嵌入：減少 API 呼叫次數 |
| Session 同步 | Session 記憶同步：對話脈絡持久化 |

**設計特點**：
- 混合搜尋結合向量與關鍵字優勢
- SQLite 本機儲存降低依賴性
- 批次最佳化降低 API 成本

---

### src/pairing/（5 個檔案）

**功能定位**：無密碼配對系統，包含 8 字元代碼產生、檔案鎖儲存、通道隔離與 TTL 過期。

| 檔案 | 功能說明 |
|------|---------|
| `code-generation.ts` | 8 字元代碼產生：隨機配對碼 |
| `file-lock-storage.ts` | 檔案鎖儲存：原子操作、防競態 |
| `channel-isolation.ts` | 通道隔離：不同通道獨立配對空間 |
| `ttl-expiry.ts` | TTL 過期：配對碼自動過期 |

**設計特點**：
- 8 字元代碼平衡安全性與易用性
- 檔案鎖確保並行安全
- TTL 過期降低安全風險

---

### src/plugins/（75+ 個檔案）

**功能定位**：外掛系統，包含外掛探索、Manifest（openclaw.plugin.json）、載入器（jiti 動態匯入）、註冊表、設定驗證與記憶體槽選擇。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `discovery/` | 外掛探索：掃描外掛目錄 |
| `manifest/` | Manifest 管理：openclaw.plugin.json 解析 |
| `loader/` | 載入器：jiti 動態匯入、熱重載 |
| `registry/` | 註冊表：外掛註冊、查詢 |
| `config-validation/` | 設定驗證：外掛設定驗證 |
| `memory-slot-selection/` | 記憶體槽選擇：外掛記憶體隔離 |

**設計特點**：
- Manifest 標準化外掛介面
- jiti 動態匯入支援 TypeScript 外掛
- 記憶體槽機制實現外掛資料隔離

---

### src/providers/（80+ 個檔案）

**功能定位**：AI 提供者整合，包含 GitHub Copilot（裝置流程認證）、Qwen（OAuth）、Google（函式呼叫驗證）與模型管理。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `github-copilot/` | GitHub Copilot：裝置流程認證 |
| `device-flow-auth/` | 裝置流程認證：OAuth 裝置授權 |
| `qwen-oauth/` | Qwen OAuth：通義千問認證 |
| `google-function-call-validation/` | Google 函式呼叫驗證：驗證 Function Calling 結構 |
| `model-management/` | 模型管理：模型註冊、查詢、切換 |

**設計特點**：
- 多提供者支援降低供應商鎖定
- 裝置流程認證適合 CLI 環境
- 函式呼叫驗證確保 AI 輸出正確性

---

### src/routing/（6 個檔案）

**功能定位**：代理路由系統，包含 Session 金鑰解析與優先權綁定（peer → guild → team → account → default）。

| 檔案 | 功能說明 |
|------|---------|
| `session-key-resolution.ts` | Session 金鑰解析：從請求解析 Session |
| `priority-binding.ts` | 優先權綁定：五層優先權機制 |

**設計特點**：
- 五層優先權（peer → guild → team → account → default）提供靈活路由
- Session 金鑰解析支援多維度路由

---

### src/security/（分散於多個檔案）

**功能定位**：安全稽核系統，多維度檢查（閘道設定、檔案權限、瀏覽器控制、日誌安全、特權執行、通道安全、機密偵測）與自動修復建議。

| 檢查面向 | 功能說明 |
|---------|---------|
| 閘道設定 | 檢查認證設定、CORS、速率限制 |
| 檔案權限 | 檢查敏感檔案權限 |
| 瀏覽器控制 | 檢查瀏覽器沙盒設定 |
| 日誌安全 | 檢查日誌是否洩漏機密 |
| 特權執行 | 檢查 sudo / 特權執行 |
| 通道安全 | 檢查通道權限設定 |
| 機密偵測 | 掃描程式碼中的硬編碼機密 |

**設計特點**：
- 多維度安全檢查涵蓋常見風險
- 自動修復建議降低人工介入
- 持續稽核確保安全基線

---

### src/sessions/（7 個檔案）

**功能定位**：Session 管理，包含對話記錄事件、Session 金鑰工具、標籤、傳送政策、模型覆寫與等級覆寫。

| 檔案 | 功能說明 |
|------|---------|
| `transcript-events.ts` | 對話記錄事件：訊息、動作、狀態變更 |
| `session-key-utils.ts` | Session 金鑰工具：金鑰產生、解析 |
| `labels.ts` | 標籤管理：Session 分類、標記 |
| `send-policy.ts` | 傳送政策：控制訊息傳送行為 |
| `model-overrides.ts` | 模型覆寫：Session 特定模型設定 |
| `level-overrides.ts` | 等級覆寫：Session 特定日誌等級 |

**設計特點**：
- 完整的 Session 生命週期管理
- 標籤系統支援 Session 分類
- 覆寫機制提供 Session 級客製化

---

### src/signal/（25+ 個檔案）

**功能定位**：Signal 通道整合，包含 CLI Daemon 管理、SSE 重連、RPC 客戶端、身份解析（UUID/E.164/username）與反應處理。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `cli-daemon-management/` | CLI Daemon 管理：signal-cli daemon 控制 |
| `sse-reconnect/` | SSE 重連：Server-Sent Events 斷線重連 |
| `rpc-client/` | RPC 客戶端：與 signal-cli 通訊 |
| `identity-resolution/` | 身份解析：UUID/E.164/username 互轉 |
| `reactions/` | 反應處理：表情符號反應 |

**設計特點**：
- signal-cli 整合實現 Signal 支援
- SSE 重連確保訊息不遺失
- 多格式身份解析提升相容性

---

### src/slack/（75+ 個檔案）

**功能定位**：Slack 通道整合，基於 @slack/bolt（Socket + HTTP 模式），包含事件訂閱、Slash Commands、執行緒解析與訊息處理。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `bolt-socket-mode/` | @slack/bolt Socket 模式：WebSocket 即時訊息 |
| `bolt-http-mode/` | @slack/bolt HTTP 模式：Webhook 訊息 |
| `event-subscription/` | 事件訂閱：訂閱 Slack 事件 |
| `slash-commands/` | Slash Commands：自訂 Slack 指令 |
| `thread-resolution/` | 執行緒解析：對話串追蹤 |
| `message-handler/` | 訊息處理：解析、路由、回覆 |

**設計特點**：
- Socket + HTTP 雙模式適應不同部署環境
- Slash Commands 提供原生 Slack 體驗
- 執行緒解析支援複雜對話結構

---

### src/telegram/（95+ 個檔案）

**功能定位**：Telegram 通道整合，基於 grammY 框架，包含 Long Polling + Webhook、循序鎖、節流器、原生指令、Markdown/HTML 格式、標題分割與內嵌鍵盤。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `grammy-framework/` | grammY 框架整合 |
| `long-polling/` | Long Polling：主動拉取訊息 |
| `webhook/` | Webhook：被動接收訊息 |
| `sequential-lock/` | 循序鎖：確保訊息順序 |
| `throttler/` | 節流器：防 API 超限 |
| `native-commands/` | 原生指令：Telegram Bot Commands |
| `markdown-format/` | Markdown 格式：MarkdownV2 格式化 |
| `html-format/` | HTML 格式：HTML 格式化 |
| `caption-splitting/` | 標題分割：長標題分段 |
| `inline-keyboards/` | 內嵌鍵盤：互動式按鈕 |

**設計特點**：
- Long Polling + Webhook 雙模式
- 循序鎖確保訊息順序處理
- 節流器避免 Telegram API 限制
- 豐富的訊息格式與互動元件

---

### src/terminal/（12 個檔案）

**功能定位**：終端機輸出工具，包含 ANSI 著色、表格、進度條、超連結、主題與串流寫入器（斷管偵測）。

| 檔案 | 功能說明 |
|------|---------|
| `ansi-coloring.ts` | ANSI 著色：彩色終端輸出 |
| `tables.ts` | 表格：格式化表格輸出 |
| `progress-bars.ts` | 進度條：視覺化進度 |
| `hyperlinks.ts` | 超連結：終端可點擊連結 |
| `theme.ts` | 主題：配色方案 |
| `stream-writer.ts` | 串流寫入器：斷管偵測 |

**設計特點**：
- 豐富的終端輸出格式
- 斷管偵測避免 pipe 錯誤
- 主題系統支援客製化配色

---

### src/tts/（約 1,580 行）

**功能定位**：文字轉語音系統，整合 Edge TTS（免費）、OpenAI（gpt-4o-mini-tts）、ElevenLabs，包含 TTS 指令解析、自動摘要與通道特定格式。

| 元件 | 功能說明 |
|------|---------|
| Edge TTS | 免費 TTS：Microsoft Edge TTS |
| OpenAI TTS | OpenAI gpt-4o-mini-tts：高品質 TTS |
| ElevenLabs | ElevenLabs：專業級 TTS |
| TTS 指令解析 | 解析 TTS 指令（語音、語速等） |
| 自動摘要 | 長文字自動摘要後轉語音 |
| 通道特定格式 | 不同通道不同音訊格式 |

**設計特點**：
- 多提供者支援從免費到專業級
- 自動摘要解決 TTS 長度限制
- 通道特定格式確保相容性

---

### src/tui/（40 個檔案）

**功能定位**：終端 UI，基於 pi-tui 的互動式聊天介面，包含 Gateway-Chat 通訊、元件（聊天記錄、編輯器、訊息渲染）、Monokai 主題、串流組裝器與 Session 動作。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `pi-tui-integration/` | pi-tui 框架整合 |
| `gateway-chat-communication/` | Gateway-Chat 通訊：與閘道通訊 |
| `components/` | UI 元件 |
| `chat-log/` | 聊天記錄：訊息歷史顯示 |
| `editor/` | 編輯器：訊息輸入 |
| `message-render/` | 訊息渲染：Markdown 渲染 |
| `monokai-theme/` | Monokai 主題：配色方案 |
| `stream-assembler/` | 串流組裝器：即時訊息組裝 |
| `session-actions/` | Session 動作：Session 操作介面 |
| `local-shell/` | 本機 Shell：整合 Shell 存取 |

**設計特點**：
- pi-tui 提供豐富的終端 UI 元件
- 串流組裝器支援即時訊息顯示
- 本機 Shell 整合提升開發體驗

---

### src/web/（100+ 個檔案）

**功能定位**：WhatsApp Web 通道整合，基於 Baileys，包含 QR 登入、Session 管理、自動回覆（含廣播 / 群組啟用 / 提及閘門）、入站媒體 / 存取控制 / 去重。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `baileys-integration/` | Baileys 整合：WhatsApp Web API |
| `qr-login/` | QR 登入：掃描 QR Code 登入 |
| `session-management/` | Session 管理：WhatsApp Session 持久化 |
| `auto-reply/` | 自動回覆系統 |
| `broadcast-activation/` | 廣播啟用：群發訊息控制 |
| `group-activation/` | 群組啟用：群組自動回覆控制 |
| `mention-gating/` | 提及閘門：僅在被提及時回覆 |
| `inbound-media/` | 入站媒體：接收圖片、影片、音訊 |
| `access-control/` | 存取控制：白名單、黑名單 |
| `deduplication/` | 去重：避免重複訊息 |

**設計特點**：
- Baileys 實現無需官方 API 的 WhatsApp 整合
- 豐富的自動回覆控制機制
- 完整的媒體處理與去重邏輯

---

## extensions/ 目錄（31 個擴充套件，596 個檔案）

### 擴充套件規模統計（按檔案數排序）

| 擴充套件名稱 | 檔案數 | 功能定位 |
|------------|-------|---------|
| `slack` | 103+ | Slack 通道進階功能擴充 |
| `telegram` | 90+ | Telegram 通道進階功能擴充 |
| `discord` | 85+ | Discord 通道進階功能擴充 |
| `llm-task` | 74 | LLM 任務編排與執行 |
| `whatsapp` | 68+ | WhatsApp 通道進階功能擴充 |
| `memory-lancedb` | 56+ | LanceDB 向量記憶體後端 |
| `zalo` | 45+ | Zalo 通道整合（越南通訊軟體） |
| `imessage` | 41+ | iMessage 通道進階功能擴充 |
| `signal` | 38+ | Signal 通道進階功能擴充 |
| `mattermost` | 35+ | Mattermost 通道整合 |

### 認證擴充套件

| 擴充套件名稱 | 功能定位 |
|------------|---------|
| `google-gemini-cli-auth` | Google Gemini CLI 認證 |
| `google-antigravity-auth` | Google Antigravity 認證 |
| `minimax-portal-auth` | Minimax Portal 認證 |
| `qwen-portal-auth` | Qwen Portal 認證 |

### 其他擴充套件

| 擴充套件名稱 | 功能定位 |
|------------|---------|
| `diagnostics-otel` | OpenTelemetry 診斷整合 |
| `copilot-proxy` | GitHub Copilot 代理 |
| `open-prose` | 開放式文字產生 |
| `feishu` | 飛書通道整合 |
| `memory-core` | 核心記憶體系統 |
| `line` | LINE 通道核心 |
| `matrix` | Matrix 通道整合 |
| `msteams` | Microsoft Teams 通道整合 |
| `nextcloud-talk` | Nextcloud Talk 通道整合 |
| `nostr` | Nostr 通道整合 |
| `tlon` | Tlon 通道整合 |
| `twitch` | Twitch 通道整合 |
| `voice-call` | 語音通話整合 |
| `bluebubbles` | BlueBubbles iMessage 橋接 |
| `googlechat` | Google Chat 通道整合 |
| `lobster` | Lobster 通道整合 |
| `zalouser` | Zalo 使用者管理 |

**設計特點**：
- 擴充套件架構實現核心與功能分離
- 通道擴充套件提供進階功能
- 認證擴充套件支援多 AI 提供者
- 記憶體擴充套件支援多後端（LanceDB、Core）

---

## apps/ 目錄（545 個檔案）

### apps/macOS/（160+ 個檔案，Swift）

**功能定位**：macOS 原生應用程式，基於 Swift，包含 Package.swift 依賴管理。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `Package.swift` | Swift Package Manager 設定 |
| `Sources/` | 原始碼目錄 |
| `Views/` | SwiftUI 視圖 |
| `Models/` | 資料模型 |
| `Services/` | 服務層（與閘道通訊） |
| `Utils/` | 工具函式 |

**設計特點**：
- 原生 macOS 應用程式提升使用體驗
- SwiftUI 實現現代化 UI
- 與閘道伺服器通訊實現功能

---

### apps/iOS/（240+ 個檔案，Swift）

**功能定位**：iOS 原生應用程式，基於 Swift，包含 Fastlane 自動化與 project.yml 專案設定。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `Fastlane/` | Fastlane 自動化：建置、測試、部署 |
| `project.yml` | XcodeGen 專案設定 |
| `Sources/` | 原始碼目錄 |
| `Views/` | SwiftUI 視圖 |
| `Models/` | 資料模型 |
| `Services/` | 服務層（與閘道通訊） |
| `Utils/` | 工具函式 |

**設計特點**：
- Fastlane 自動化建置與部署流程
- XcodeGen 產生 Xcode 專案，避免合併衝突
- 與 macOS 應用共用部分程式碼

---

### apps/Android/（120+ 個檔案，Kotlin）

**功能定位**：Android 原生應用程式，基於 Kotlin，使用 Gradle 建置系統。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `build.gradle` | Gradle 建置設定 |
| `app/` | 主要應用程式模組 |
| `src/main/` | 原始碼目錄 |
| `java/` | Kotlin/Java 原始碼 |
| `res/` | 資源檔（佈局、圖片、字串） |
| `manifests/` | AndroidManifest.xml |

**設計特點**：
- Kotlin 優先開發
- Material Design UI
- 與閘道伺服器通訊實現功能

---

### apps/shared/（25+ 個檔案，OpenClawKit）

**功能定位**：跨平台共用程式碼，OpenClawKit 框架。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `Models/` | 共用資料模型 |
| `Networking/` | 網路通訊層 |
| `Utils/` | 共用工具函式 |

**設計特點**：
- 跨平台共用減少程式碼重複
- OpenClawKit 提供統一 API
- Swift Package 形式供各平台匯入

---

## scripts/ 目錄（93 個檔案）

**功能定位**：建置、打包、測試、協定產生、文件國際化、同步與監控腳本。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `build/` | 建置腳本 |
| `package-mac-app/` | macOS 應用程式打包 |
| `package-dmg/` | DMG 映像檔產生 |
| `code-signing/` | macOS 程式碼簽署 |
| `docker/` | Docker 相關腳本 |
| `e2e/` | E2E 測試腳本 |
| `sandbox/` | 沙盒測試腳本 |
| `test/` | 測試工具腳本 |
| `protocol-generation/` | 協定檔案產生（gRPC、A2UI） |
| `docs-i18n/` | 文件國際化（Go） |
| `sync/` | 同步腳本 |
| `monitor/` | 監控腳本 |

**設計特點**：
- 完整的自動化建置流程
- 多平台打包與簽署
- 文件國際化自動化

---

## skills/ 目錄（52 個技能，75 個檔案）

**功能定位**：AI 代理技能庫，包含密碼管理、筆記、天氣、GitHub 整合、編碼代理等。

| 技能名稱 | 功能說明 |
|---------|---------|
| `1password` | 1Password 整合：密碼管理 |
| `apple-notes` | Apple Notes 整合：筆記存取 |
| `weather` | 天氣查詢 |
| `github` | GitHub 整合：Issue、PR 管理 |
| `coding-agent` | 編碼代理：程式碼產生、審查 |
| 其他 47 個技能 | 各種專業領域技能 |

**設計特點**：
- 模組化技能系統
- 技能可獨立安裝與啟用
- 豐富的預建技能庫

---

## ui/ 目錄（142 個檔案）

**功能定位**：Vite + Vitest Web 前端，包含應用程式模組與聊天元件。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `vite.config.ts` | Vite 建置設定 |
| `vitest.config.ts` | Vitest 測試設定 |
| `src/` | 原始碼目錄 |
| `app-modules/` | 應用程式模組 |
| `chat-components/` | 聊天 UI 元件 |
| `components/` | 通用 UI 元件 |
| `stores/` | 狀態管理（Pinia / Zustand） |
| `router/` | 路由設定 |

**設計特點**：
- Vite 提供快速開發體驗
- 元件化設計提升可維護性
- 狀態管理實現複雜互動

---

## test/ 目錄（16 個檔案）

**功能定位**：測試全域設定、輔助工具與 E2E 測試。

| 檔案 | 功能說明 |
|------|---------|
| `global-setup.ts` | 全域測試設定 |
| `helpers/` | 測試輔助工具 |
| `e2e/` | E2E 測試 |
| `gateway.e2e.test.ts` | 閘道伺服器 E2E 測試 |
| `media.e2e.test.ts` | 媒體處理 E2E 測試 |
| `provider-timeout.e2e.test.ts` | 提供者逾時 E2E 測試 |

**設計特點**：
- 全域設定統一測試環境
- E2E 測試涵蓋關鍵流程
- 輔助工具簡化測試撰寫

---

## docs/ 目錄（665 個檔案）

**功能定位**：英文 + 簡體中文文件，依概念、安裝、啟動、通道、平台、閘道、CLI、外掛、提供者、鉤子、工具、自動化、參考、安全、除錯、診斷分類。

| 子目錄 | 功能說明 | 檔案數估計 |
|-------|---------|----------|
| `concepts/` | 概念說明：架構、設計理念 | 50+ |
| `install/` | 安裝指南：各平台安裝步驟 | 30+ |
| `start/` | 快速開始：新手教學 | 20+ |
| `channels/` | 通道文件：各通道設定 | 80+ |
| `platforms/` | 平台文件：各平台特性 | 40+ |
| `gateway/` | 閘道文件：閘道伺服器設定 | 50+ |
| `cli/` | CLI 文件：指令參考 | 60+ |
| `plugins/` | 外掛文件：外掛開發指南 | 70+ |
| `providers/` | 提供者文件：AI 提供者設定 | 50+ |
| `hooks/` | 鉤子文件：鉤子開發指南 | 40+ |
| `tools/` | 工具文件：工具使用指南 | 30+ |
| `automation/` | 自動化文件：自動化設定 | 30+ |
| `reference/` | 參考文件：API 參考 | 60+ |
| `security/` | 安全文件：安全最佳實踐 | 20+ |
| `debug/` | 除錯文件：除錯技巧 | 15+ |
| `diagnostics/` | 診斷文件：診斷工具使用 | 20+ |

**設計特點**：
- 雙語文件（英文 + 簡體中文）
- 結構化組織便於查找
- 涵蓋從新手到專家的所有需求

---

## vendor/ 目錄（173 個檔案）

**功能定位**：A2UI 渲染引擎，包含渲染器與規格說明。

| 子目錄 / 檔案 | 功能說明 |
|-------------|---------|
| `renderers/` | 渲染器實作：Canvas、DOM、Native |
| `specification/` | A2UI 規格說明：協定定義 |

**設計特點**：
- A2UI 協定實現跨平台 UI 渲染
- 多渲染器支援不同執行環境
- Vendor 目錄隔離第三方程式碼

---

## packages/ 目錄（6 個檔案）

**功能定位**：子套件，包含 clawdbot、moltbot。

| 子套件 | 功能說明 |
|-------|---------|
| `clawdbot` | Clawdbot 機器人套件 |
| `moltbot` | Moltbot 機器人套件 |

**設計特點**：
- Monorepo 子套件管理
- 獨立發布與版本控制

---

## Swabble/ 目錄

**功能定位**：Swift 6.2 喚醒詞 Daemon，支援 macOS 26+，使用 Speech.framework 與 AVAudioEngine 管線。

| 元件 | 功能說明 |
|------|---------|
| Speech.framework | 語音辨識框架 |
| AVAudioEngine | 音訊引擎：麥克風輸入、處理 |
| Daemon | 背景執行的喚醒詞偵測服務 |

**設計特點**：
- Swift 6.2 現代化語法
- macOS 26+ 系統框架整合
- 低延遲喚醒詞偵測

---

## 總結

OpenClaw 專案採用 **Monorepo 架構**，包含以下主要組成部分：

1. **核心系統**（`src/`）：2,606 個檔案，涵蓋 AI 代理、訊息處理、瀏覽器自動化、多通道整合、記憶體系統等
2. **擴充套件系統**（`extensions/`）：31 個擴充套件，596 個檔案，提供通道進階功能與 AI 提供者整合
3. **原生應用程式**（`apps/`）：545 個檔案，涵蓋 macOS、iOS、Android 與跨平台共用程式碼
4. **自動化腳本**（`scripts/`）：93 個檔案，處理建置、打包、測試、文件國際化
5. **技能庫**（`skills/`）：52 個技能，75 個檔案，提供專業領域能力
6. **Web UI**（`ui/`）：142 個檔案，基於 Vite 的現代化前端
7. **文件**（`docs/`）：665 個檔案，雙語完整文件
8. **A2UI 渲染引擎**（`vendor/`）：173 個檔案，跨平台 UI 協定
9. **子套件**（`packages/`）：6 個檔案，機器人套件
10. **喚醒詞 Daemon**（`Swabble/`）：Swift 6.2 語音喚醒服務

**整體設計特點**：
- **模組化架構**：各子系統職責單一，鬆散耦合
- **跨平台支援**：Web、macOS、iOS、Android 全平台覆蓋
- **擴充性強**：外掛系統、鉤子系統、技能系統支援第三方擴充
- **多通道整合**：支援 10+ 個通訊平台（Discord、Slack、Telegram、WhatsApp 等）
- **AI 優先**：深度整合多個 AI 提供者，提供豐富的 AI 能力
- **安全性**：多維度安全檢查、沙盒執行、權限控制
- **開發者友善**：完整文件、豐富的 CLI 工具、自動化測試

---

**文件版本**：v1.0
**最後更新**：2026-02-08
**對應專案版本**：OpenClaw 2026.2.6+
