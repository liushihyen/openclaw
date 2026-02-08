# OpenClaw — 建置、部署與基礎設施分析

> **階段 5**：Build, Deployment, and Infrastructure Analysis
> 從原始碼到上線的完整管線，以及多平台部署策略

---

## 執行摘要

OpenClaw 採用現代化的建置與部署架構，核心特色包含：

- **TypeScript 編譯管線**：使用 `tsdown` 進行多進入點打包，支援 ESM/CJS 雙模式相容
- **容器化優先**：提供三種 Dockerfile（主服務、沙箱、瀏覽器沙箱），支援 Docker Compose 一鍵部署
- **多平台 CI/CD**：GitHub Actions 驅動的矩陣測試（Linux、Windows、macOS）與雙架構映像建置（amd64/arm64）
- **雲端部署就緒**：內建 Fly.io 與 Render 設定檔，支援公開與私有部署模式
- **嚴格品質閘道**：Pre-commit hooks 整合 10+ 種檢查工具，Protocol Conformance 自動化驗證

---

## 1. 建置管線（Build Pipeline）

### 1.1 主建置指令：`pnpm build`

完整建置流程包含 8 個步驟，依序執行：

```bash
# package.json scripts 定義
{
  "build": "pnpm canvas:a2ui:bundle && tsdown && pnpm build:plugin-sdk:dts && node --import tsx scripts/write-plugin-sdk-entry-dts.ts && node --import tsx scripts/canvas-a2ui-copy.ts && node --import tsx scripts/copy-hook-metadata.ts && node --import tsx scripts/write-build-info.ts && node --import tsx scripts/write-cli-compat.ts"
}
```

#### 步驟解析

| 步驟 | 指令 | 檔案位置 | 用途 |
|------|------|---------|------|
| 1 | `pnpm canvas:a2ui:bundle` | `scripts/bundle-a2ui.sh` | 打包 Canvas A2UI 前端資產（React UI 元件） |
| 2 | `tsdown` | `tsdown.config.ts` | TypeScript 編譯與打包（4 個進入點） |
| 3 | `pnpm build:plugin-sdk:dts` | `tsconfig.plugin-sdk.dts.json` | 產生 Plugin SDK 的型別宣告檔（.d.ts） |
| 4 | `node --import tsx scripts/write-plugin-sdk-entry-dts.ts` | `scripts/write-plugin-sdk-entry-dts.ts` | 撰寫 Plugin SDK 進入點的型別定義 |
| 5 | `node --import tsx scripts/canvas-a2ui-copy.ts` | `scripts/canvas-a2ui-copy.ts` | 複製 A2UI 編譯產出到 `dist/` |
| 6 | `node --import tsx scripts/copy-hook-metadata.ts` | `scripts/copy-hook-metadata.ts` | 複製 Hook 中繼資料（用於擴充功能系統） |
| 7 | `node --import tsx scripts/write-build-info.ts` | `scripts/write-build-info.ts` | 寫入建置資訊（版本號、Git commit hash、建置時間） |
| 8 | `node --import tsx scripts/write-cli-compat.ts` | `scripts/write-cli-compat.ts` | 產生 CLI 相容層（支援舊版 `openclaw` 指令） |

### 1.2 tsdown 打包設定

位置：`tsdown.config.ts`

OpenClaw 使用 [tsdown](https://github.com/egoist/tsdown)（Rust 驅動的 TypeScript 打包工具）處理四個進入點：

```typescript
// tsdown.config.ts 核心設定
export default defineConfig([
  {
    // 1. 主函式庫進入點
    entry: 'src/index.ts',
    outDir: 'dist',
    format: ['esm'],
    platform: 'node',
    dts: false,  // 型別宣告由獨立流程產生
    env: { NODE_ENV: 'production' }
  },
  {
    // 2. CLI 進入點（openclaw 指令）
    entry: 'src/entry.ts',
    outDir: 'dist',
    format: ['esm'],
    platform: 'node',
    dts: false,
    env: { NODE_ENV: 'production' }
  },
  {
    // 3. Plugin SDK（供第三方擴充使用）
    entry: 'src/plugin-sdk/index.ts',
    outDir: 'dist/plugin-sdk',
    format: ['esm'],
    platform: 'node',
    dts: false,
    env: { NODE_ENV: 'production' }
  },
  {
    // 4. Extension API（瀏覽器擴充介面）
    entry: 'src/extensionAPI.ts',
    outDir: 'dist',
    format: ['esm'],
    platform: 'node',
    dts: false,
    env: { NODE_ENV: 'production' }
  }
])
```

#### 進入點功能說明

| 進入點 | 產出路徑 | 用途 | 暴露對象 |
|--------|---------|------|---------|
| `src/index.ts` | `dist/index.js` | 主函式庫進入點 | Node.js 應用程式 |
| `src/entry.ts` | `dist/entry.js` | CLI 指令進入點 | Shell 環境（`#!/usr/bin/env node`） |
| `src/plugin-sdk/index.ts` | `dist/plugin-sdk/index.js` | Plugin SDK | 第三方擴充開發者 |
| `src/extensionAPI.ts` | `dist/extensionAPI.js` | Extension API | 瀏覽器擴充功能（Chrome/Firefox） |

### 1.3 TypeScript 編譯設定

位置：`tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2023",                    // 目標語法：ES2023（支援 Top-level await）
    "module": "NodeNext",                  // 模組系統：ESM/CJS 自動相容
    "moduleResolution": "NodeNext",
    "strict": true,                        // 嚴格模式：全面啟用型別檢查
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "experimentalDecorators": true,        // 支援裝飾器（用於 MCP Server）
    "noEmit": true,                        // 不產生 JS（由 tsdown 處理）
    "paths": {
      "src/*": ["src/*"],
      "ui/*": ["ui/*"]
    }
  },
  "include": ["src/**/*", "ui/**/*"],
  "exclude": ["node_modules", "dist", "build"]
}
```

**關鍵設計決策**：

1. **NodeNext 模組系統**：支援 ESM（`import`）與 CJS（`require`）雙模式，確保與不同 Node.js 版本相容
2. **noEmit: true**：TypeScript 僅負責型別檢查，實際編譯由 `tsdown` 執行（效能更好）
3. **experimentalDecorators**：啟用裝飾器語法，用於 MCP Server 的路由定義（`@rpc` 裝飾器）

---

## 2. 測試架構（Testing Framework）

### 2.1 Vitest 設定矩陣

OpenClaw 採用分層測試策略，針對不同測試類型提供專用設定檔：

| 設定檔 | 測試範圍 | Worker 數量 | Timeout | 備註 |
|--------|---------|------------|---------|------|
| `vitest.config.ts` | 主測試（單元 + 整合） | CI: 3, 本地: auto | 120s | 預設設定，排除 Gateway 與 Extension |
| `vitest.unit.config.ts` | 單元測試 | auto | 120s | 僅測試純邏輯模組 |
| `vitest.gateway.config.ts` | Gateway Server | auto | 120s | 測試 HTTP Gateway 與 WebSocket |
| `vitest.extensions.config.ts` | 擴充系統 | auto | 120s | 測試 Plugin SDK 與 Hook 機制 |
| `vitest.e2e.config.ts` | 端對端測試 | CI: 2, 本地: 1-4 | 120s | 完整使用者流程模擬 |
| `vitest.live.config.ts` | Live API 測試 | 1（循序執行） | 120s | 真實 API 呼叫，避免平行競爭 |
| `ui/vitest.config.ts` | 瀏覽器測試 | - | - | 使用 Playwright，測試 Canvas UI |

#### 測試覆蓋率門檻

```typescript
// vitest.config.ts
coverage: {
  thresholds: {
    lines: 70,        // 行覆蓋率 ≥ 70%
    functions: 70,    // 函式覆蓋率 ≥ 70%
    branches: 55,     // 分支覆蓋率 ≥ 55%
    statements: 70    // 陳述式覆蓋率 ≥ 70%
  }
}
```

#### 隔離策略

```typescript
// 使用 forks pool（行程隔離）
pool: 'forks'  // 確保測試間無狀態污染
```

### 2.2 測試指令清單

```bash
# 平行單元測試（使用 scripts/test-parallel.mjs 管理）
pnpm test

# 完整測試套件（Lint → Build → Test → E2E → Live → Docker）
pnpm test:all

# 端對端測試
pnpm test:e2e

# Live API 測試（需設定 OPENCLAW_LIVE_TEST=1）
pnpm test:live

# Docker 隔離測試（容器內執行完整測試）
pnpm test:docker:all
```

### 2.3 測試平行化機制

位置：`scripts/test-parallel.mjs`

```javascript
// 平行執行策略
const testGroups = [
  { config: 'vitest.config.ts', workers: 3 },
  { config: 'vitest.gateway.config.ts', workers: 2 },
  { config: 'vitest.extensions.config.ts', workers: 2 }
]

// CI 環境自動調整 Worker 數量
const workers = process.env.CI
  ? Math.min(group.workers, 3)  // CI 限制資源使用
  : 'auto'                       // 本地環境自動偵測
```

---

## 3. 容器化（Containerization）

### 3.1 主服務容器：Dockerfile

位置：`Dockerfile`

#### 建置策略

```dockerfile
# 階段 1：基礎映像
FROM node:22-bookworm as base
WORKDIR /app

# 安裝 Bun + pnpm（透過 corepack）
RUN curl -fsSL https://bun.sh/install | bash && \
    corepack enable && \
    corepack prepare pnpm@latest --activate

# 階段 2：依賴安裝
FROM base as deps
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# 階段 3：建置
FROM deps as builder
COPY . .
RUN pnpm build

# 階段 4：執行時映像
FROM base as runner
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# 安全性：使用非 root 使用者
USER node

# 預設綁定 localhost（防止意外暴露）
ENV OPENCLAW_BIND=127.0.0.1

# 啟動 Gateway Server（允許未設定狀態）
CMD ["node", "openclaw.mjs", "gateway", "--allow-unconfigured"]
```

#### 關鍵設計決策

1. **多階段建置（Multi-stage Build）**：最終映像僅包含執行時檔案，減少映像大小約 60%
2. **USER node**：以 UID 1000 的非特權使用者執行，符合安全最佳實踐
3. **預設 localhost 綁定**：避免容器在未設定情況下意外暴露於公網
4. **--allow-unconfigured**：允許首次啟動時產生預設設定（簡化 Docker Compose 使用）

### 3.2 沙箱容器：Dockerfile.sandbox

位置：`Dockerfile.sandbox`

```dockerfile
FROM debian:bookworm-slim

# 最小化依賴：僅安裝工具執行必要套件
RUN apt-get update && apt-get install -y \
    bash \
    ca-certificates \
    curl \
    git \
    jq \
    python3 \
    ripgrep \
    && rm -rf /var/lib/apt/lists/*

# 保持容器運作，等待 exec 指令
CMD ["sleep", "infinity"]
```

**用途**：為 Bash Tool 提供隔離執行環境，防止使用者程式碼影響主系統。

### 3.3 瀏覽器沙箱容器：Dockerfile.sandbox-browser

位置：`Dockerfile.sandbox-browser`

```dockerfile
FROM openclaw-sandbox

# 安裝 Chromium + X11 + VNC
RUN apt-get update && apt-get install -y \
    chromium \
    xvfb \           # 虛擬 Framebuffer（無頭模式）
    x11vnc \         # VNC Server
    novnc \          # Web VNC Client
    && rm -rf /var/lib/apt/lists/*

# 暴露埠
EXPOSE 9222  # Chrome DevTools Protocol
EXPOSE 5900  # VNC
EXPOSE 6080  # noVNC Web Interface

# 啟動 Xvfb + Chromium + VNC
CMD ["bash", "-c", "Xvfb :99 & chromium --remote-debugging-port=9222 & x11vnc -display :99 -forever & websockify --web=/usr/share/novnc 6080 localhost:5900"]
```

**用途**：支援瀏覽器自動化測試（Playwright、Puppeteer），提供遠端除錯介面。

### 3.4 Docker Compose 編排

位置：`docker-compose.yml`

```yaml
services:
  # Gateway Server（生產模式）
  openclaw-gateway:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "18789:18789"  # Gateway HTTP
      - "18790:18790"  # Bridge WebSocket
    volumes:
      - ~/.openclaw:/home/node/.openclaw  # 持久化設定與工作區
    environment:
      - NODE_ENV=production
      - OPENCLAW_STATE_DIR=/home/node/.openclaw
    restart: unless-stopped

  # CLI 互動模式（開發用）
  openclaw-cli:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ~/.openclaw:/home/node/.openclaw
    stdin_open: true
    tty: true
    entrypoint: ["node", "openclaw.mjs"]
    command: ["--help"]
```

### 3.5 Docker 自動化部署腳本

位置：`docker-setup.sh`

```bash
#!/bin/bash
# 自動化部署流程

# 步驟 1：檢查依賴（Docker、Docker Compose）
check_dependencies() {
  command -v docker >/dev/null || { echo "Docker not found"; exit 1; }
  command -v docker-compose >/dev/null || { echo "Docker Compose not found"; exit 1; }
}

# 步驟 2：初始化設定目錄
init_config() {
  mkdir -p ~/.openclaw/{config,workspace,logs}
}

# 步驟 3：產生安全 Token
generate_token() {
  openssl rand -hex 32 > ~/.openclaw/config/gateway-token
}

# 步驟 4：建置映像
docker-compose build

# 步驟 5：引導設定（Onboarding）
docker-compose run --rm openclaw-cli onboard

# 步驟 6：設定 Channel（Telegram、Discord、Slack）
docker-compose run --rm openclaw-cli channel add

# 步驟 7：啟動 Gateway
docker-compose up -d openclaw-gateway
```

---

## 4. 雲端部署平台

### 4.1 Fly.io（公開部署）

位置：`fly.toml`

```toml
app = "openclaw"
primary_region = "iad"  # 可設定為其他區域（如 nrt、sin、lhr）

[build]
  dockerfile = "Dockerfile"

[env]
  NODE_ENV = "production"
  OPENCLAW_PREFER_PNPM = "true"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = false
  auto_start_machines = true
  min_machines_running = 1

[[vm]]
  size = "shared-cpu-2x"
  memory = "2048mb"

[mounts]
  source = "openclaw_data"
  destination = "/data"

[processes]
  app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"
```

#### 關鍵設定

| 設定項 | 值 | 說明 |
|--------|-----|------|
| `primary_region` | `iad` | 主要部署區域（Virginia），可改為 `nrt`（Tokyo）等 |
| `vm.size` | `shared-cpu-2x` | 共享 CPU，2 核心 |
| `vm.memory` | `2048mb` | 2GB RAM（Gateway 建議最低值） |
| `auto_stop_machines` | `false` | 保持機器運作（避免冷啟動延遲） |
| `min_machines_running` | `1` | 最少 1 台機器（確保服務可用） |
| `force_https` | `true` | 強制 HTTPS（安全性） |
| `--bind lan` | - | 綁定內部網路介面（Fly.io 自動處理外部路由） |

#### 部署指令

```bash
# 登入 Fly.io
fly auth login

# 建立應用程式（首次部署）
fly launch --config fly.toml

# 設定 API Key
fly secrets set OPENAI_API_KEY=sk-xxx ANTHROPIC_API_KEY=sk-ant-xxx

# 部署
fly deploy

# 查看日誌
fly logs

# 擴充容量（增加機器數量）
fly scale count 2
```

### 4.2 Fly.io（私有部署）

位置：`fly.private.toml`

```toml
app = "openclaw-private"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

# 無 [http_service] 區塊 → 不暴露公開網路

[[vm]]
  size = "shared-cpu-2x"
  memory = "2048mb"

[mounts]
  source = "openclaw_data"
  destination = "/data"
```

#### 私有部署存取方式

```bash
# 方式 1：Fly Proxy（本地轉發）
fly proxy 18789:3000 -a openclaw-private

# 方式 2：WireGuard VPN（Fly.io 私有網路）
fly wireguard create

# 方式 3：SSH Tunnel
fly ssh console -a openclaw-private
```

**適用情境**：
- 僅需對外 API 呼叫（OpenAI、Anthropic），無需接受外部請求
- 透過 Telegram Bot、Discord Bot 等主動連線的 Channel
- 企業內網部署（搭配 VPN）

### 4.3 Render

位置：`render.yaml`

```yaml
services:
  - type: web
    name: openclaw-gateway
    runtime: docker
    plan: starter  # 或 standard、pro
    dockerfilePath: ./Dockerfile
    envVars:
      - key: NODE_ENV
        value: production
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true  # 自動產生隨機 Token
    healthCheckPath: /health
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
    autoDeploy: true
```

#### 部署流程

1. 連結 GitHub Repository
2. Render 自動偵測 `render.yaml`
3. 首次部署後，在 Dashboard 設定環境變數（API Keys）
4. 每次 Push 到 `main` 自動重新部署

---

## 5. CI/CD 管線

### 5.1 主 CI 管線：GitHub Actions

位置：`.github/workflows/ci.yml`

#### 觸發條件

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: ['**']  # 所有 PR
```

#### Job 矩陣

```mermaid
graph TB
    A[Push/PR Trigger] --> B[install-check]
    B --> C1[checks: tsgo]
    B --> C2[checks: lint]
    B --> C3[checks: test]
    B --> C4[checks: protocol]
    B --> C5[checks: format]
    B --> D[secrets]
    B --> E1[checks-windows: build]
    B --> E2[checks-windows: lint]
    B --> E3[checks-windows: test]
    B --> E4[checks-windows: protocol]
    B -.PR Only.-> F[macos]

    C3 --> G[Upload Coverage]
    F --> H1[TS Tests]
    F --> H2[Swift Checks]
```

#### Job 詳細說明

| Job | 執行環境 | 用途 | 關鍵指令 |
|-----|---------|------|---------|
| `install-check` | Ubuntu (4vCPU) | 依賴驗證 | `pnpm install --frozen-lockfile` |
| `checks` (matrix) | Ubuntu | 程式碼品質矩陣 | tsgo, lint, test, protocol, format |
| `secrets` | Ubuntu | 秘密掃描 | `detect-secrets scan` |
| `checks-windows` | Windows | Windows 相容性 | 同 `checks` 但在 Windows 上執行 |
| `macos` | macOS | macOS + Swift 檢查 | TS 測試 + SwiftLint |

#### 關鍵設定

```yaml
# install-check job
runs-on: ubuntu-latest-4-cores  # 使用 4 核心（加速 pnpm install）

env:
  NODE_VERSION: '22.x'
  PNPM_VERSION: '10.23.0'

# checks job（矩陣）
strategy:
  matrix:
    check: [tsgo, lint, test, protocol, format]
  fail-fast: false  # 一個失敗不中斷其他檢查

# 快取策略
- uses: actions/setup-node@v4
  with:
    node-version: ${{ env.NODE_VERSION }}
    cache: 'pnpm'
```

### 5.2 Docker 映像發布管線

位置：`.github/workflows/docker-release.yml`

#### 觸發條件

```yaml
on:
  push:
    branches: [main]
    tags: ['v*']
```

#### 雙架構建置

```yaml
platforms: linux/amd64,linux/arm64

# 使用 QEMU 模擬 ARM64
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
```

#### Tag 策略

```yaml
# main 分支 → latest
tags: ghcr.io/openclaw/openclaw:latest

# v* tag → 版本號
tags: ghcr.io/openclaw/openclaw:1.2.3
```

#### 快取策略

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

**說明**：使用 GitHub Actions Cache，大幅加速後續建置（首次建置 ~8 分鐘，後續 ~2 分鐘）。

### 5.3 Protocol Conformance 檢查

位置：`.github/workflows/formal-conformance.yml`

#### 用途

驗證 Gateway 與 Client 之間的 Protocol 一致性，防止版本漂移（Drift）。

#### 測試套件

```yaml
# Green Suite（正常情境，40+ 檢查）
- Precedence tests（優先順序）
- Group tests（群組功能）
- Gateway auth（閘道認證）
- Remote MCP（遠端 MCP）
- Streaming（串流處理）
- Error handling（錯誤處理）

# Negative Suite（預期失敗情境）
- Invalid auth（無效認證）
- Malformed requests（格式錯誤請求）
- Resource exhaustion（資源耗盡）
```

#### 漂移偵測

```yaml
- name: Check for drift
  if: failure()
  run: |
    echo "::error::Protocol drift detected! Client and Gateway implementations are out of sync."
    echo "Run 'pnpm test:protocol' locally to investigate."
```

### 5.4 安裝腳本驗證

位置：`.github/workflows/install-smoke.yml`

```yaml
# 測試情境
- Ubuntu 22.04 + bash
- macOS 14 + zsh
- Windows Server 2022 + PowerShell

# 驗證項目
1. 腳本下載與執行
2. 依賴安裝（Node.js、pnpm）
3. openclaw 指令可執行
4. openclaw --version 正確回傳
```

---

## 6. 程式碼品質工具

### 6.1 Pre-commit Hooks 鏈

位置：`.pre-commit-config.yaml`

```yaml
repos:
  # 1. 基礎檔案檢查
  - repo: https://github.com/pre-commit/pre-commit-hooks
    hooks:
      - trailing-whitespace   # 移除行尾空白
      - end-of-file-fixer     # 確保檔案結尾有換行
      - check-yaml            # YAML 語法檢查
      - check-added-large-files  # 防止大檔案誤加入

  # 2. 秘密掃描
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

  # 3. Shell Script 檢查
  - repo: https://github.com/koalaman/shellcheck-precommit
    hooks:
      - id: shellcheck
        args: ['--severity=error']

  # 4. GitHub Actions 檢查
  - repo: https://github.com/rhysd/actionlint
    rev: v1.7.10
    hooks:
      - id: actionlint

  # 5. GitHub Actions 安全稽核
  - repo: https://github.com/woodruffw/zizmor
    rev: v1.22.0
    hooks:
      - id: zizmor

  # 6. TypeScript/JavaScript Linting（oxlint）
  - repo: local
    hooks:
      - id: oxlint
        name: oxlint
        entry: pnpm exec oxlint --type-aware src test
        language: system
        pass_filenames: false

  # 7. 程式碼格式化（oxfmt）
  - repo: local
    hooks:
      - id: oxfmt
        name: oxfmt
        entry: pnpm exec oxfmt --check src test
        language: system
        pass_filenames: false

  # 8. Swift Linting
  - repo: https://github.com/realm/SwiftLint
    hooks:
      - id: swiftlint
        args: ['--config', '.swiftlint.yml']

  # 9. Swift Formatting
  - repo: https://github.com/nicklockwood/SwiftFormat
    hooks:
      - id: swiftformat
        args: ['--lint']
```

### 6.2 Linting 工具詳解

#### oxlint（Rust-based ESLint 替代方案）

```bash
# 基本檢查
pnpm exec oxlint src test

# 型別感知檢查（使用 TypeScript 型別資訊）
pnpm exec oxlint --type-aware src test

# 自動修復
pnpm exec oxlint --fix src test
```

**優勢**：
- 速度比 ESLint 快 50-100 倍（Rust 實作）
- 支援 `--type-aware`（型別檢查）
- 內建常用規則，零設定啟動

#### oxfmt（Rust-based Prettier 替代方案）

```bash
# 檢查格式
pnpm exec oxfmt --check src test

# 自動格式化
pnpm exec oxfmt src test
```

**優勢**：
- 速度比 Prettier 快 20-50 倍
- 相容 Prettier 設定檔（`.prettierrc`）
- 支援 TypeScript、JavaScript、JSON、Markdown

#### shellcheck（Shell 腳本靜態分析）

```bash
# 檢查所有 .sh 檔案
shellcheck scripts/*.sh

# 僅顯示 error 級別問題
shellcheck --severity=error scripts/*.sh
```

**常見規則**：
- SC2086：變數未加引號（可能造成字串分割）
- SC2046：使用 `$(...)` 取代反引號
- SC2164：`cd` 失敗後繼續執行（應加 `|| exit`）

#### zizmor（GitHub Actions 安全稽核）

```bash
# 掃描所有 Workflows
zizmor .github/workflows/

# 僅顯示高風險問題
zizmor --severity high .github/workflows/
```

**檢查項目**：
- 未固定版本的 Action（應使用 `@sha` 而非 `@v1`）
- Secret 洩漏風險（`echo $SECRET` 等）
- 權限過度開放（`permissions: write-all`）
- Injection 攻擊風險（`${{ github.event.issue.title }}` 未跳脫）

---

## 7. 環境設定管理

### 7.1 環境變數模板

位置：`.env.example`

```bash
# ===== API Keys =====
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=...
GOOGLE_AI_STUDIO_API_KEY=...

# ===== Gateway 設定 =====
OPENCLAW_GATEWAY_TOKEN=...           # Gateway 認證 Token（建議使用 openssl rand -hex 32 產生）
OPENCLAW_GATEWAY_PORT=18789          # Gateway HTTP 埠
OPENCLAW_GATEWAY_BIND=127.0.0.1      # 綁定介面（0.0.0.0 允許外部存取）

# ===== Channel 設定 =====
TELEGRAM_BOT_TOKEN=...               # Telegram Bot Token
DISCORD_BOT_TOKEN=...                # Discord Bot Token
SLACK_BOT_TOKEN=xoxb-...             # Slack Bot Token
SLACK_SIGNING_SECRET=...             # Slack 簽章驗證

# ===== Twilio（語音通話） =====
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_NUMBER=+1234567890

# ===== 資料庫（選用） =====
DATABASE_URL=postgresql://user:pass@localhost:5432/openclaw

# ===== Redis（選用，快取） =====
REDIS_URL=redis://localhost:6379

# ===== 日誌設定 =====
LOG_LEVEL=info                       # debug | info | warn | error
LOG_FORMAT=json                      # json | pretty

# ===== 沙箱設定 =====
OPENCLAW_SANDBOX_IMAGE=openclaw-sandbox  # Docker 沙箱映像名稱
OPENCLAW_PREFER_SANDBOX=true             # 優先使用沙箱執行工具

# ===== 開發設定 =====
NODE_ENV=development                 # development | production
OPENCLAW_STATE_DIR=~/.openclaw       # 狀態目錄（設定檔、日誌、工作區）
NODE_OPTIONS=--max-old-space-size=4096  # Node.js 記憶體限制（4GB）
```

### 7.2 Docker 環境變數

```yaml
# docker-compose.yml
environment:
  - NODE_ENV=production
  - OPENCLAW_PREFER_PNPM=true
  - OPENCLAW_STATE_DIR=/home/node/.openclaw
  - NODE_OPTIONS=--max-old-space-size=4096
```

### 7.3 執行時環境變數載入順序

```
1. 系統環境變數（Shell export）
   ↓
2. .env 檔案（dotenv 載入）
   ↓
3. CLI 參數（--env-file 指定）
   ↓
4. 程式碼內硬編碼預設值
```

**優先順序**：系統環境變數 > .env 檔案 > CLI 參數 > 預設值

---

## 8. 常駐服務管理

### 8.1 macOS（LaunchAgent）

位置：`~/Library/LaunchAgents/com.anthropic.openclaw.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.anthropic.openclaw</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/node</string>
        <string>/usr/local/lib/node_modules/@anthropic-ai/openclaw/dist/entry.js</string>
        <string>gateway</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/你的使用者名稱/.openclaw/logs/stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/你的使用者名稱/.openclaw/logs/stderr.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>OPENAI_API_KEY</key>
        <string>sk-...</string>
        <key>OPENCLAW_GATEWAY_TOKEN</key>
        <string>...</string>
    </dict>
</dict>
</plist>
```

#### 管理指令

```bash
# 載入服務
launchctl load ~/Library/LaunchAgents/com.anthropic.openclaw.plist

# 卸載服務
launchctl unload ~/Library/LaunchAgents/com.anthropic.openclaw.plist

# 檢查狀態
launchctl list | grep openclaw

# 查看日誌
tail -f ~/.openclaw/logs/stdout.log
```

### 8.2 Linux（systemd User Unit）

位置：`~/.config/systemd/user/openclaw-gateway.service`

```ini
[Unit]
Description=OpenClaw Gateway Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/node /usr/local/lib/node_modules/@anthropic-ai/openclaw/dist/entry.js gateway
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# 環境變數
Environment="NODE_ENV=production"
Environment="OPENCLAW_STATE_DIR=%h/.openclaw"
EnvironmentFile=%h/.openclaw/config/env

[Install]
WantedBy=default.target
```

#### 管理指令

```bash
# 啟用服務（開機自動啟動）
systemctl --user enable openclaw-gateway.service

# 啟動服務
systemctl --user start openclaw-gateway.service

# 檢查狀態
systemctl --user status openclaw-gateway.service

# 查看日誌
journalctl --user -u openclaw-gateway.service -f

# 重啟服務
systemctl --user restart openclaw-gateway.service
```

### 8.3 Windows（Task Scheduler）

#### PowerShell 腳本

```powershell
# 建立排程工作
$action = New-ScheduledTaskAction -Execute "node" -Argument "C:\Program Files\nodejs\node_modules\@anthropic-ai\openclaw\dist\entry.js gateway"
$trigger = New-ScheduledTaskTrigger -AtStartup
$principal = New-ScheduledTaskPrincipal -UserId "$env:USERNAME" -LogonType Interactive
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries

Register-ScheduledTask -TaskName "OpenClaw Gateway" -Action $action -Trigger $trigger -Principal $principal -Settings $settings
```

#### 管理指令

```powershell
# 啟動工作
Start-ScheduledTask -TaskName "OpenClaw Gateway"

# 停止工作
Stop-ScheduledTask -TaskName "OpenClaw Gateway"

# 移除工作
Unregister-ScheduledTask -TaskName "OpenClaw Gateway" -Confirm:$false
```

---

## 9. 原生應用程式建置

### 9.1 macOS（Swift）

位置：`Package.swift`

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "OpenClaw",
    platforms: [
        .macOS(.v14)  // 要求 macOS 14+
    ],
    products: [
        .executable(name: "openclaw", targets: ["OpenClaw"])
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-argument-parser", from: "1.3.0")
    ],
    targets: [
        .executableTarget(
            name: "OpenClaw",
            dependencies: [
                .product(name: "ArgumentParser", package: "swift-argument-parser")
            ]
        )
    ]
)
```

#### 建置指令

```bash
# 開發模式建置
swift build

# 發布模式建置（最佳化）
swift build -c release

# 執行
.build/release/openclaw --version

# 安裝到系統
sudo cp .build/release/openclaw /usr/local/bin/
```

### 9.2 iOS（xcodegen + Fastlane）

位置：`project.yml`（xcodegen 設定）

```yaml
name: OpenClaw
options:
  bundleIdPrefix: com.anthropic
targets:
  OpenClaw:
    type: application
    platform: iOS
    deploymentTarget: "17.0"
    sources:
      - path: Sources
    settings:
      PRODUCT_BUNDLE_IDENTIFIER: com.anthropic.openclaw
```

#### 建置流程

```bash
# 1. 產生 Xcode 專案
xcodegen generate

# 2. 使用 Fastlane 建置
bundle exec fastlane ios build

# 3. 或直接使用 xcodebuild
xcodebuild -scheme OpenClaw -configuration Release -destination 'platform=iOS Simulator,name=iPhone 15'
```

### 9.3 Android（Gradle）

位置：`app/build.gradle.kts`

```kotlin
android {
    namespace = "com.anthropic.openclaw"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.anthropic.openclaw"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}
```

#### 建置指令

```bash
# 開發版本（Debug）
./gradlew :app:assembleDebug

# 發布版本（Release）
./gradlew :app:assembleRelease

# 安裝到裝置
./gradlew :app:installDebug

# 執行測試
./gradlew :app:testDebugUnitTest
```

---

## 10. CI/CD 管線總覽圖

```mermaid
graph TB
    A[Git Push/PR] --> B{Trigger Type}

    B -->|Push to main| C1[ci.yml]
    B -->|Tag v*| C2[docker-release.yml]
    B -->|Any PR| C3[formal-conformance.yml]
    B -->|Install script change| C4[install-smoke.yml]

    C1 --> D1[install-check]
    D1 --> E1[checks Matrix]
    E1 --> F1[tsgo]
    E1 --> F2[lint]
    E1 --> F3[test]
    E1 --> F4[protocol]
    E1 --> F5[format]

    D1 --> E2[checks-windows]
    E2 --> G1[build]
    E2 --> G2[lint]
    E2 --> G3[test]
    E2 --> G4[protocol]

    D1 --> E3[macos]
    E3 --> H1[TS Tests]
    E3 --> H2[Swift Checks]

    D1 --> E4[secrets]
    E4 --> I1[detect-secrets]

    C2 --> J1[Docker Buildx]
    J1 --> J2[linux/amd64]
    J1 --> J3[linux/arm64]
    J2 --> K[Push to GHCR]
    J3 --> K

    C3 --> L1[Green Suite]
    C3 --> L2[Negative Suite]
    L1 --> M{All Pass?}
    L2 --> M
    M -->|Yes| N[Success]
    M -->|No| O[Drift Detected]

    C4 --> P1[Ubuntu Test]
    C4 --> P2[macOS Test]
    C4 --> P3[Windows Test]

    style C1 fill:#e1f5ff
    style C2 fill:#fff4e1
    style C3 fill:#f0e1ff
    style C4 fill:#e1ffe4
    style K fill:#4caf50
    style N fill:#4caf50
    style O fill:#f44336
```

---

## 11. 建置與部署最佳實踐建議

### 11.1 本地開發環境

```bash
# 1. 安裝依賴（使用 pnpm）
pnpm install

# 2. 設定環境變數
cp .env.example .env
# 編輯 .env，填入 API Keys

# 3. 啟動開發模式（熱重載）
pnpm dev

# 4. 執行測試（平行）
pnpm test

# 5. 建置（檢查產出是否正常）
pnpm build
```

### 11.2 Docker 本地測試

```bash
# 1. 建置映像
docker build -t openclaw:local .

# 2. 執行容器
docker run -it --rm \
  -p 18789:18789 \
  -v ~/.openclaw:/home/node/.openclaw \
  -e OPENAI_API_KEY=sk-... \
  openclaw:local

# 3. 進入容器除錯
docker run -it --rm openclaw:local /bin/bash
```

### 11.3 生產環境部署檢查清單

- [ ] 環境變數已正確設定（API Keys、Tokens）
- [ ] `NODE_ENV=production` 已設定
- [ ] 日誌輸出導向檔案或日誌服務（避免 stdout 過載）
- [ ] Health Check 端點正常回應（`/health`）
- [ ] 資料庫 Migration 已執行
- [ ] 監控與告警已設定（Prometheus、Sentry）
- [ ] 自動重啟機制已啟用（Docker restart policy、systemd Restart=always）
- [ ] HTTPS 已啟用（Fly.io force_https、Nginx SSL）
- [ ] 速率限制已設定（防止 API 濫用）
- [ ] 備份策略已建立（資料庫、設定檔）

### 11.4 效能最佳化建議

1. **減少 Docker 映像大小**：
   - 使用 `node:22-alpine` 替代 `node:22-bookworm`（減少 ~200MB）
   - `.dockerignore` 排除 `node_modules/.cache`、`*.log`

2. **啟用 HTTP/2**：
   ```typescript
   // src/gateway/server.ts
   import http2 from 'http2'
   const server = http2.createSecureServer(options, app)
   ```

3. **使用 Redis 快取**：
   ```typescript
   // 快取 API 回應
   const cachedResult = await redis.get(`result:${key}`)
   if (cachedResult) return JSON.parse(cachedResult)
   ```

4. **啟用 Compression**：
   ```typescript
   import compression from 'compression'
   app.use(compression())
   ```

---

## 12. 故障排除

### 12.1 常見問題

#### 問題 1：Docker 建置失敗（`pnpm install` 卡住）

**原因**：網路問題或 Registry 鏡像未設定

**解決方式**：
```dockerfile
# Dockerfile 加入 pnpm config
RUN pnpm config set registry https://registry.npmmirror.com
```

#### 問題 2：Gateway 啟動後無法連線

**檢查項目**：
```bash
# 1. 檢查埠是否綁定
netstat -tuln | grep 18789

# 2. 檢查防火牆
sudo ufw status

# 3. 檢查 Docker 網路
docker network inspect bridge

# 4. 查看 Gateway 日誌
docker logs openclaw-gateway
```

#### 問題 3：CI 測試逾時

**原因**：Worker 數量過高，資源競爭

**解決方式**：
```yaml
# vitest.config.ts
pool: 'forks',
poolOptions: {
  forks: {
    maxForks: process.env.CI ? 2 : 4  // CI 限制 Worker 數量
  }
}
```

### 12.2 監控與日誌

#### 推薦工具

| 用途 | 工具 | 整合方式 |
|------|------|---------|
| 應用程式監控 | Sentry | `@sentry/node` |
| 效能監控 | Prometheus + Grafana | `/metrics` 端點 |
| 日誌聚合 | Loki + Grafana | Docker logging driver |
| 錯誤追蹤 | Sentry | 自動捕捉未處理例外 |

#### 日誌設定範例

```typescript
// src/utils/logger.ts
import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }  // 開發模式：美化輸出
    : undefined                  // 生產模式：JSON 格式
})
```

---

## 附錄：指令速查表

### 建置相關

| 指令 | 用途 |
|------|------|
| `pnpm build` | 完整建置（TS 編譯 + UI 打包） |
| `pnpm build:watch` | 監聽模式建置 |
| `pnpm clean` | 清除 dist/ 與 node_modules/.cache |

### 測試相關

| 指令 | 用途 |
|------|------|
| `pnpm test` | 平行單元測試 |
| `pnpm test:e2e` | E2E 測試 |
| `pnpm test:live` | Live API 測試 |
| `pnpm test:docker:all` | Docker 隔離測試 |

### Docker 相關

| 指令 | 用途 |
|------|------|
| `docker-compose up -d` | 背景啟動服務 |
| `docker-compose logs -f` | 查看即時日誌 |
| `docker-compose exec openclaw-gateway sh` | 進入容器 Shell |
| `docker-compose down` | 停止並移除容器 |

### 雲端部署

| 指令 | 用途 |
|------|------|
| `fly deploy` | 部署到 Fly.io |
| `fly logs` | 查看 Fly.io 日誌 |
| `fly ssh console` | SSH 連線到 Fly.io 機器 |
| `git push render main` | 部署到 Render（自動） |

---

**文件版本**：1.0.0
**最後更新**：2026-02-08
**維護者**：OpenClaw 架構分析團隊
