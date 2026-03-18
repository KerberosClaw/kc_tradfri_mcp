# kc_tradfri_mcp

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.12+-blue.svg)](https://python.org)
[![FastMCP](https://img.shields.io/badge/FastMCP-3.x-orange.svg)](https://github.com/jlowin/fastmcp)
[![MCP](https://img.shields.io/badge/Protocol-MCP-purple.svg)](https://modelcontextprotocol.io)

IKEA TRADFRI 智慧家居的 MCP Server。將 CoAP-over-DTLS 通訊封裝成 MCP tools，讓 AI assistant 能用自然語言控制燈具、插座與場景。

MCP Server for IKEA TRADFRI smart home gateway. Wraps CoAP-over-DTLS into MCP tools so AI assistants can control lights, plugs, and scenes via natural language.

> **詳細教程 / Full tutorial**：[`docs/openclaw-tradfri-mcp-tutorial.md`](docs/openclaw-tradfri-mcp-tutorial.md)
> **DTLS 踩坑紀錄 / DTLS pitfalls**：[`docs/dtls-tradfri-pitfalls.md`](docs/dtls-tradfri-pitfalls.md)

---

## 架構 / Architecture

```
User (Telegram / Web UI)
  → AI Agent (OpenClaw / Claude Desktop / etc.)
  → mcporter CLI (MCP client)
  → tradfri-mcp (Docker, FastMCP HTTP server, port 8765)
  → aiocoap (CoAP over DTLS)
  → TRADFRI gateway (LAN, UDP 5684)
  → Zigbee → 燈具 / Lights
```

## 目錄結構 / Project Structure

```
kc_tradfri_mcp/
├── server.py              # FastMCP HTTP server（主程式）
├── coap_client.py         # aiocoap 封裝（CoAP GET/PUT，singleton context）
├── config.py              # 環境變數設定
├── devices.py             # 設備拓撲管理（devices.json / aliases.json）
├── aliases.json           # 自訂別名（含 virtual room）
├── devices.json           # 設備快取（scan 產生，.gitignore）
├── .tradfri_psk.json      # PSK 憑證（.gitignore）
├── .env / .env.example    # 環境變數
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml / uv.lock
├── vendor/dtlssocket/     # DTLSSocket 0.2.3（TinyDTLS 已 patch）
├── scripts/
│   ├── gen_psk.py         # 產生 PSK 憑證
│   └── scan.py            # 掃描 gateway 設備
├── openclaw-skill/        # OpenClaw skill（見「OpenClaw 整合」）
│   ├── SKILL.md
│   ├── _meta.json
│   ├── .clawhub/origin.json
│   └── scripts/
│       └── tradfri        # wrapper script（簡化 mcporter 呼叫）
└── docs/
    ├── dtls-tradfri-pitfalls.md
    └── openclaw-tradfri-mcp-tutorial.md
```

---

## 快速開始 / Quick Start

### 1. Clone 並安裝依賴

```bash
git clone https://github.com/YOUR_USERNAME/kc_tradfri_mcp.git
cd kc_tradfri_mcp
uv sync
```

### 2. 產生 PSK 憑證

```bash
export TRADFRI_GATEWAY_IP=192.168.x.x
export TRADFRI_SECURITY_CODE=xxxxxxxxxxxxxxxx   # gateway 背面的碼

uv run python scripts/gen_psk.py
# → .tradfri_psk.json 建立完成
```

> 若已有 `.tradfri_psk.json`，跳過此步驟。
> DTLS 相關問題見 [`docs/dtls-tradfri-pitfalls.md`](docs/dtls-tradfri-pitfalls.md)。

### 3. 掃描設備

```bash
uv run python scripts/scan.py
# 輸出存為 devices.json；或啟動後用 mcporter call tradfri.refresh_devices
```

### 4. 設定

```bash
cp .env.example .env
# 編輯 .env，填入 TRADFRI_GATEWAY_IP
```

編輯 `aliases.json`，設定自訂名稱（支援四種類型）：

```json
{
  "客廳": {
    "type": "virtual",
    "groups": [131079],
    "devices": [65545, 65553, 65554, 65555]
  },
  "主臥室": {"type": "group", "id": 131089},
  "餐廳軌道燈": {"type": "device_list", "ids": [65579, 65580, 65581]},
  "餐桌燈": {"type": "device", "id": 65551}
}
```

| 類型 | 說明 |
|------|------|
| `virtual` | 虛擬房間：組合多個 IKEA group + 獨立設備 |
| `group` | IKEA 原生群組 |
| `device_list` | 多個設備的集合（無 IKEA group 時用）|
| `device` | 單一設備 |

### 5. 啟動（Docker）

```bash
docker compose up -d
docker compose logs -f   # 確認 DTLS 握手成功
```

### 6. 連接 mcporter

```bash
npm install -g mcporter
mcporter config add tradfri --url http://localhost:8765/mcp
mcporter list tradfri   # 確認 tools 出現
```

### 7. 測試

```bash
mcporter call tradfri.list_aliases
mcporter call tradfri.control_by_name name=客廳 state=false
mcporter call tradfri.control_by_name name=客廳 state=true
mcporter call tradfri.set_color_temp name=餐桌燈 direction=warm
```

---

## MCP Tools

| Tool | 說明 |
|------|------|
| `control_group` | 控制群組（開關、亮度）|
| `control_device` | 控制單一設備 |
| `control_by_name` | **最常用** — 以 alias 名稱控制（支援所有 alias 類型）|
| `set_color_temp` | 色溫調整（`direction: warm/cool` 或 `mireds: 250-454`）|
| `set_color` | 設定顏色（RGB 燈泡：red/green/blue/orange/yellow/purple/pink）|
| `activate_scene` | 觸發場景 |
| `get_status` | 查詢即時狀態（支援 `name=`）|
| `list_devices` | 列出所有設備、群組、場景、alias |
| `list_aliases` | 列出 alias 清單（輕量，供 LLM 快速確認）|
| `refresh_devices` | 重新掃描 gateway，更新 devices.json |
| `find_by_name` | 名稱 → ID 解析 |
| `send_notification` | Telegram 推播（未設定時靜默略過）|

---

## 環境變數 / Environment Variables

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `TRADFRI_GATEWAY_IP` | **是** | — | Gateway LAN IP |
| `MCP_PORT` | | `8765` | HTTP server port |
| `MCP_HOST` | | `0.0.0.0` | Bind address |
| `PSK_FILE` | | `.tradfri_psk.json` | PSK 憑證路徑 |
| `DEVICES_FILE` | | `devices.json` | 設備快取路徑 |
| `ALIASES_FILE` | | `aliases.json` | 別名對應路徑 |
| `TELEGRAM_BOT_TOKEN` | | — | Telegram Bot token（選填） |
| `TELEGRAM_CHAT_ID` | | — | Telegram Chat ID（選填） |
| `TRADFRI_POLL_INTERVAL` | | `30` | OBSERVE 中斷後重試間隔（秒）|

---

## OpenClaw 整合

### 原理

OpenClaw 沒有 `mcpServers` 設定（與 Claude Desktop 不同）。它的 MCP 整合走 **mcporter skill**：AI agent 透過 `exec` 工具執行 `mcporter` CLI 來呼叫 MCP tools。

但 8B 模型（如 qwen3-vl:8b-instruct）無法可靠地 exec 複雜的 mcporter 語法：

```bash
# 這對 8B 模型太複雜 — 多個 key=value 參數 + 不同 tool 名稱
mcporter call tradfri.control_by_name name=客廳 state=true
mcporter call tradfri.set_color_temp name=客廳 direction=warm
```

解法是 **SearXNG 模式**：建一個 wrapper script，把複雜度藏起來，讓 LLM 只需 exec 簡單的中文指令：

```bash
tradfri 客廳 開
tradfri 客廳 關
tradfri 客廳 亮度 80       # 百分比 0-100
tradfri 客廳 色溫 暖
tradfri 餐桌燈 顏色 紅     # RGB 燈泡：紅/綠/藍/橙/黃/紫/粉
tradfri 查詢 客廳
tradfri 列表
```

### 安裝步驟

**前置：** 確認 mcporter 已安裝且已設定（見「快速開始」步驟 6）。

**1. 安裝 OpenClaw skill**

```bash
# 複製到 OpenClaw workspace（不能用 symlink，OpenClaw 會拒絕跨目錄的 realPath）
cp -r openclaw-skill ~/.openclaw/workspace/skills/tradfri
```

**2. 安裝 wrapper script**

```bash
ln -s $(pwd)/openclaw-skill/scripts/tradfri /opt/homebrew/bin/tradfri
# Linux：ln -s $(pwd)/openclaw-skill/scripts/tradfri /usr/local/bin/tradfri
```

**3. 在 AGENTS.md 加入燈控指令**

在 `~/.openclaw/workspace/AGENTS.md` 加入（**不是 systemPrompt，不是 SKILL.md**）：

```markdown
## IKEA TRADFRI 燈控

收到燈控請求 → 立即 exec `tradfri` 指令，不解釋不確認。

tradfri 客廳電視牆 開
tradfri 客廳 關
tradfri 客廳 亮度 80
tradfri 餐桌燈 色溫 暖
tradfri 餐桌燈 顏色 紅
tradfri 查詢 沙發燈
tradfri 列表
```

> **重要：** 只有 AGENTS.md 的內容會被完整注入到 LLM context。systemPrompt 和 SKILL.md 不可靠。

**4. 重啟 OpenClaw gateway**

```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

**5. 測試**

在 Telegram 對 OpenClaw 說「開客廳燈」，確認燈亮起來。

### SearXNG 模式說明

此整合方案參考 SearXNG 的成功模式：

| | SearXNG | TRADFRI |
|---|---------|---------|
| wrapper | `/opt/homebrew/bin/searxng-search` | `/opt/homebrew/bin/tradfri` |
| 實際執行 | `uv run searxng.py search "$@"` | `mcporter call tradfri.* ...` |
| LLM 看到的 | `searxng-search "關鍵字"` | `tradfri 客廳 開` |
| 複雜度 | 一個引號參數 | 中文名稱 + 中文動作 |
| skill | `searxng/SKILL.md` + triggers | `tradfri/SKILL.md` + triggers |

---

## 踩坑紀錄 / Pitfalls

### Docker `network_mode: host` 在 macOS 不能用

macOS Docker Desktop 跑在 LinuxKit VM 裡，`network_mode: host` 只暴露 VM 的網路，不是 Mac 的 LAN。

**解法：** 使用預設 bridge 網路 + `ports` 映射。bridge 網路可以透過 VM NAT 存取 LAN IP（包括 gateway 的 UDP 5684）。此方案在 macOS 和 Linux 都能用。

```yaml
# docker-compose.yml
services:
  tradfri-mcp:
    ports:
      - "8765:8765"     # 不要用 network_mode: host
```

### CoAP context 的所有權：OBSERVE 負責 reset

原始設計中 `coap_put` / `coap_get` 失敗時會 `_ctx = None` 重置 context。但這會破壞正在運作的 OBSERVE session（因為 TRADFRI gateway 每個 PSK identity 只允許一個 DTLS session）。

**正確做法：**
- `coap_put` / `coap_get` 失敗時 **不 reset context**，直接 raise
- OBSERVE task 偵測到連線中斷後，才呼叫 `reset_ctx()` 清除舊 context
- 下次 `get_ctx()` 時自動建立新的 DTLS session

這個設計讓控制操作的暫時失敗不會影響正在運作的 OBSERVE 訂閱。

### OBSERVE 不需要 semaphore 序列化

曾經嘗試用 `asyncio.Semaphore(1)` + 2 秒延遲來序列化 OBSERVE 初始化，認為 20 個並發 GET 會壓垮 gateway。實測證明 **不需要**：gateway 可以處理並發 OBSERVE GET，問題根源是上面的 context reset bug，不是並發量。

移除 semaphore 後 OBSERVE 初始化時間從 ~40 秒降到幾秒。

### OpenClaw skill 不能用 symlink

`~/.openclaw/workspace/skills/tradfri` 如果是 symlink 指向 repo 目錄，OpenClaw 會拒絕載入：`Skipping skill path that resolves outside its configured root.`。必須用 `cp -r` 複製實際檔案。

### OpenClaw 只有 AGENTS.md 會被完整注入 LLM context

`openclaw.json` 的 `systemPrompt` 被放在 system prompt 末尾，容易被截斷或被模型忽略。`SKILL.md` 只有 name/description 被引用，內容不注入。只有 `AGENTS.md` 的內容完整出現在 LLM 看到的 system prompt 中。所有關鍵指令（燈控、搜尋）都要寫在 AGENTS.md。

### aliases.json 的 `_comment` 會導致 list_devices crash

`aliases.json` 裡的 `"_comment": "..."` 是 string，`list_devices` 裡 `target.get("type")` 會 crash。修法：跳過非 dict 的 entries。

### 8B LLM 無法可靠 exec 複雜 mcporter 語法

qwen3-vl:8b-instruct 在收到「開客廳燈」時嘗試 exec `mcporter call tradfri.control_by_name name=客廳 state=true`，但結果是：
- PC 風扇狂轉（模型 inference 卡住），最終無回應
- 或模型產生幻覺（捏造結果，實際沒有 exec）

**根因：** 多個 `key=value` 參數 + 不同 tool 名稱對 8B 模型太複雜。

**解法：** 仿照 SearXNG 成功模式，建 wrapper script 讓 LLM 只需 exec 簡化的中文指令。

### systemPrompt 不要塞 TRADFRI 細節

8B 模型的 context window 有限（49K tokens，但 tool schemas 已佔用大量空間）。把 21 個設備名稱 + 5 種指令格式全塞進 systemPrompt 會：
- 增加 inference 時間
- 降低其他指令（搜尋、SSH、cron）的遵循度

**解法：** systemPrompt 只留一句觸發規則，細節放在 skill SKILL.md。Skill 只在 trigger 匹配時才注入 context。

---

## 常見問題 / Troubleshooting

| 錯誤 | 解法 |
|------|------|
| `CredentialsMissingError` | credentials URI 去掉 `:5684` port — 見 [`dtls-tradfri-pitfalls.md` #9](docs/dtls-tradfri-pitfalls.md) |
| DTLS 握手失敗 | 需 patch TinyDTLS C 原始碼 — 見 [`dtls-tradfri-pitfalls.md` #6 #7](docs/dtls-tradfri-pitfalls.md) |
| `NetworkError` 循環失敗 | 確認 `coap_client.py` 的 `coap_put`/`coap_get` 沒有 `_ctx = None`（見上方踩坑紀錄）|
| 設備找不到 | `mcporter call tradfri.refresh_devices` 或 `uv run python scripts/scan.py` |
| mcporter 連不上 | `docker compose ps` 確認容器運作，`curl http://localhost:8765/mcp` 確認 HTTP |
| Docker 容器無法連 gateway | macOS 不支援 `network_mode: host`，改用 bridge + `ports`（見上方踩坑紀錄）|

---

## 本機開發 / Development (without Docker)

```bash
TRADFRI_GATEWAY_IP=192.168.x.x uv run python server.py

# MCP Inspector (Web UI)
npx @modelcontextprotocol/inspector http://localhost:8765/mcp
# → http://localhost:6274
```

---

## License

MIT
