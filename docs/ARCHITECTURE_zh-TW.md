# 架構說明：hermes-agent

> 由 `/map` 於 2026-05-20 產生，涵蓋提交 `7c2ff742a`。

## 概覽

Hermes Agent 是一個多模態 AI 代理執行環境，透過互動式 CLI、訊息平台（Telegram、Discord、Slack、WhatsApp、Signal 等 10 個以上）及透過 Agent Communication Protocol（ACP）的 IDE 整合，將大型語言模型連接至使用者。它解決了從單一程式碼庫將具備工具呼叫能力的 AI 代理部署至多元介面（聊天應用、終端機、排程任務、遠端編輯器）的問題。技術棧以 Python（後端執行環境）為主，搭配 React/TypeScript（Web 儀表板與 TUI），並透過 FastAPI、SQLite 及可插拔的 provider/plugin 架構協調運作。

---

## 技術棧

| 層次 | 技術 |
|------|------|
| 語言 | Python 3.11+（執行環境）、TypeScript/React（TUI 與 Web 儀表板） |
| LLM 提供者 | Anthropic、OpenAI、Google Gemini、AWS Bedrock、Azure AI Foundry、DeepSeek 及其他 25+ 個 |
| CLI / TUI | prompt_toolkit（REPL）、Ink/React via Node.js（TUI）、FastAPI（儀表板後端） |
| 閘道器 | python-telegram-bot、discord.py、slack-bolt、mautrix 及各平台專屬 SDK |
| 資料庫 | SQLite（WAL 模式）— 會話狀態、看板、排程任務 |
| 測試 | pytest（50+ 個測試檔案） |
| 協定 | ACP v0.9.0（IDE/編輯器整合）、MCP v1.26.0（工具擴充） |
| 依賴管理 | uv + 精確版本鎖定的 `pyproject.toml`（供應鏈強化） |
| 基礎設施 / 部署 | Docker、Nix flake、Homebrew、Modal/Daytona/Vercel（無伺服器運算選項） |

---

## 目錄結構

```
hermes-agent/
├── agent/                    # 核心 AI 代理執行環境（對話迴圈、介面卡、工具）
│   ├── lsp/                  # Language Server Protocol 客戶端整合
│   ├── transports/           # 各提供者專屬格式轉換（基底類別 + 實作）
│   ├── agent_init.py         # AIAgent.__init__（約 1,400 行）：提供者偵測、初始化
│   ├── conversation_loop.py  # run_conversation()：每輪對話的主驅動（約 3,900 行）
│   ├── context_compressor.py # 自動上下文視窗壓縮
│   ├── context_engine.py     # 抽象 ContextEngine 介面
│   ├── memory_manager.py     # 記憶提供者協調（同時僅支援一個外部插件）
│   ├── tool_executor.py      # 並發工具排程（ThreadPoolExecutor）
│   ├── skill_commands.py     # CLI 與閘道器共用的 /斜線指令處理器
│   ├── curator.py            # 背景技能生命週期維護（獨立進程）
│   ├── web_search_registry.py# Web 搜尋提供者註冊與備援鏈
│   ├── anthropic_adapter.py  # Anthropic SDK 傳輸層
│   ├── bedrock_adapter.py    # AWS Bedrock 傳輸層（boto3、過期連線處理）
│   ├── gemini_native_adapter.py # Google Gemini 傳輸層
│   ├── azure_identity_adapter.py# Azure Foundry + Entra ID 傳輸層
│   └── chat_completion_helpers.py # 前綴快取標頭、串流、Token 計算
├── tools/                    # 工具實作，在工具註冊表中登錄
│   ├── registry.py           # 中央工具註冊表（AST 掃描，避免循環引入）
│   ├── lazy_deps.py          # 允許清單控管的執行期依賴安裝器
│   ├── mcp_tool.py           # MCP 協定工具橋接
│   └── ...                   # 程式碼執行、瀏覽器自動化、沙箱、憑證等
├── gateway/                  # 多平台訊息閘道器常駐程序
│   ├── platforms/            # 各平台介面卡（telegram/、discord/、slack/、matrix/ 等）
│   │   └── base.py           # PlatformAdapter 抽象基底類別
│   ├── run.py                # GatewayRunner：生命週期管理、LRU 代理快取（128 個會話，TTL 1 小時）
│   └── delivery.py           # 依平台分割訊息並傳遞
├── hermes_cli/               # CLI 入口點、設定、認證、技能瀏覽器、排程 UI、儀表板
│   ├── main.py               # cmd_chat()：頂層入口；路由至 TUI 或 CLI REPL
│   └── auth.py               # OAuth 憑證管理
├── cli.py                    # prompt_toolkit REPL（3,000+ 行）、代理初始化、會話 DB
├── run_agent.py              # AIAgent 外觀類別（委派至 agent/ 模組）
├── model_tools.py            # 工具註冊表協調層；擁有長駐 asyncio 事件迴圈
├── cron/                     # 排程任務執行器
│   └── scheduler.py          # 每 60 秒輪詢；以靜默模式啟動 AIAgent；日誌寫至 ~/.hermes/cron/
├── plugins/                  # 可選的模組化擴充（以 pip 套件形式安裝）
│   ├── model-providers/      # 額外的 LLM 提供者插件（openai-codex/、anthropic/、bedrock/ 等）
│   ├── web/                  # Web 搜尋後端（brave、ddg、searxng、firecrawl、exa 等）
│   ├── memory/               # 外部記憶提供者整合（honcho、mem0 等）
│   ├── image-gen/            # 圖片生成插件（fal.ai 等）
│   ├── video-gen/            # 影片生成插件
│   ├── observability/        # 追蹤與指標插件
│   ├── platforms/            # 額外訊息平台插件
│   └── dashboard/            # 儀表板擴充插件
├── providers/                # 宣告式提供者設定檔系統（~/.hermes/active_profile）
├── acp_adapter/              # Agent Communication Protocol 伺服器，用於 IDE 整合
│   ├── server.py             # ACP 伺服器（以 ThreadPoolExecutor 包裝同步 AIAgent 呼叫）
│   ├── session.py            # ACP 會話狀態、工具集擴充、檢查點/回滾
│   └── __main__.py           # ACP 伺服器入口點
├── skills/                   # 打包的官方技能（資料檔案，非可引入模組）
│   └── 分類/技能名稱/SKILL.md
├── optional-skills/          # 預設未啟用的官方技能
├── acp_registry/             # 技能登錄表中繼資料
├── ui-tui/                   # 終端機 UI（Ink/React，Node.js）
├── tui_gateway/              # 閘道器常駐程序與 TUI 渲染層的整合橋接
├── web/                      # React/TypeScript Web 儀表板（SPA + FastAPI 後端）
├── tests/                    # pytest 測試套件（50+ 個檔案）
├── scripts/                  # 建置、安裝及工具腳本
├── locales/                  # 國際化翻譯（20+ 種語言）
├── docs/                     # 文件與規劃文件
├── docker/                   # Docker 設定
├── packaging/                # Homebrew formula
├── nix/                      # Nix flake 定義
├── datagen-config-examples/  # 設定範例檔案
├── website/                  # 文件網站原始碼
├── hermes_state.py           # 基於 ContextVar 的每任務家目錄作用域
└── pyproject.toml            # 精確版本鎖定依賴（供應鏈強化）
```

---

## 目錄角色與慣例

### `/agent`
**角色：** 核心 AI 代理執行環境——從接收使用者輸入到回傳回應之間發生的一切：對話迴圈、上下文管理、工具執行、模型傳輸、記憶體與技能。
**慣例：** 每個 LLM 提供者有自己的 `*_adapter.py` 檔案。傳輸邏輯（格式轉換）放在 `agent/transports/`，不放在介面卡本身。新的上下文引擎必須實作 `ContextEngine` 抽象基底類別。**不得**將工具實作放在此處——那屬於 `tools/`。

### `/tools`
**角色：** 代理在對話中可呼叫的所有工具實作。每個工具檔案在模組引入時透過 `registry.register()` 自我註冊。
**慣例：** 必須在模組層級呼叫 `registry.register()`，讓 `registry.py` 的 AST 掃描器在不執行引入的情況下進行探索。執行期可安裝的依賴必須透過 `lazy_deps.py` 的允許清單宣告——任意 pip 安裝會被阻擋。**不得**在此加入業務邏輯或 LLM 通訊。

### `/gateway`
**角色：** 長駐常駐程序，橋接訊息平台（Telegram、Discord、Slack、Matrix 等）至代理執行環境。一個 `GatewayRunner` 實例管理所有平台；每個會話的 `AIAgent` 實例透過 LRU 快取（最多 128 個，閒置 TTL 1 小時）。
**慣例：** 每個平台放在 `gateway/platforms/{platform_name}/`。所有平台必須繼承 `gateway/platforms/base.py` 的 `PlatformAdapter`。訊息傳遞（依平台限制分割）集中在 `gateway/delivery.py`——**不得**在平台介面卡內部進行分割。

### `/hermes_cli`
**角色：** CLI 入口點及面向使用者的工具：設定精靈、憑證管理、技能瀏覽器、排程 UI，以及支援 Web UI 的儀表板伺服器。
**慣例：** `main.py:cmd_chat()` 是所有互動式會話的授權入口點。設定檔選擇（`--profile/-p`）必須在所有其他模組引入之前在此處理。

### `/plugins`
**角色：** 以獨立 pip 套件形式發布的可選擴充。涵蓋額外的 LLM 提供者、記憶體後端、Web 搜尋後端、圖片/影片生成、可觀測性，以及額外的訊息平台。
**慣例：** 記憶體提供者必須是獨立的 pip 套件——不再接受納入程式碼庫中。插件啟用透過提供者設定檔系統管理，而非直接引入。每個插件子目錄可獨立安裝。

### `/cron`
**角色：** 排程自動化引擎。背景執行緒每 60 秒輪詢一次，識別到期任務，並以不需互動的靜默模式 `AIAgent` 執行（工具自動批准）。
**慣例：** 排程任務輸出寫入 `~/.hermes/cron/outputs/{job_id}.log`。任務訊息加上 `[SILENT]` 前綴可抑制平台傳遞，但仍會寫入本地日誌。任務狀態（排程、下次執行時間）持久化至 SQLite。

### `/acp_adapter`
**角色：** 透過 Agent Communication Protocol（ACP v0.9.0）對外暴露 Hermes Agent，用於 IDE 與編輯器整合（例如 VS Code）。
**慣例：** ACP 是同步的——伺服器使用 `ThreadPoolExecutor` 包裝同步的 `AIAgent` 呼叫。會話狀態、工具集擴充及檢查點/回滾均放在 `acp_adapter/session.py`。

### `/skills`
**角色：** 打包的官方技能定義，以資料檔案形式發布（Markdown + shell/API 包裝器）。組織結構為 `skills/分類/技能名稱/SKILL.md`。
**慣例：** 技能是 Markdown 檔案，不是可引入的 Python 模組。透過 `setup.py` 以資料檔案形式打包。新的程式碼庫內建技能放在此處；`optional-skills/` 用於官方支援但非預設的技能。**不得**將 Python 工具實作放在 `skills/` 中。

### `/providers`
**角色：** 宣告式提供者設定檔系統。設定檔定義給定上下文中哪個推論提供者和模型處於啟用狀態；當前設定檔追蹤於 `~/.hermes/active_profile`。
**慣例：** 切換設定檔必須透過此系統——不得在代理或閘道器程式碼中硬編碼提供者選擇。

### `/ui-tui`
**角色：** 以 Ink（React for terminals）驅動的終端機 UI，以 Node.js 子進程運行。處理即時 Token 串流顯示。
**慣例：** TUI 在 Token 串流時即時渲染；CLI REPL（`cli.py`）則等待完整回應。這是兩條獨立的渲染路徑——**不得**將它們合併。

### `/tui_gateway`
**角色：** 訊息閘道器常駐程序與 TUI 渲染層之間的整合橋接。

### `/tests`
**角色：** pytest 測試套件，涵蓋代理執行環境、閘道器、CLI、工具及整合測試。
**慣例：** Windows 專屬測試（如 `gateway-windows`）必須透過猴子補丁（例如 `ctypes.windll`）相容非 Windows 環境。

### `/web`
**角色：** 本地儀表板的 React/TypeScript 單頁應用程式，由 `hermes_cli` 啟動的 FastAPI 伺服器作為後端。

---

## 關鍵模組

| 模組 | 路徑 | 角色 |
|------|------|------|
| AIAgent（外觀） | `run_agent.py` | 頂層代理類別；委派初始化與對話至 `agent/` 模組 |
| init_agent | `agent/agent_init.py` | AIAgent 初始化：提供者自動偵測、憑證解析、上下文引擎設定（約 1,400 行） |
| run_conversation | `agent/conversation_loop.py` | 驅動單輪對話：模型呼叫、工具排程、重試、備援、壓縮、記憶體提示（約 3,900 行） |
| ContextCompressor | `agent/context_compressor.py` | 使用輔助模型進行不可逆的中間輪次摘要；實作 ContextEngine |
| ContextEngine | `agent/context_engine.py` | 可插拔上下文引擎的抽象基底類別 |
| MemoryManager | `agent/memory_manager.py` | 協調記憶體提供者；強制單一外部插件限制 |
| Tool Registry | `tools/registry.py` | 透過 AST 掃描探索的中央工具註冊表；避免循環引入 |
| model_tools | `model_tools.py` | 工具註冊表協調層；擁有單一長駐 asyncio 事件迴圈 |
| execute_tool_calls_concurrent | `agent/tool_executor.py` | 透過 ThreadPoolExecutor 並發工具排程，具備平行作用域偵測 |
| GatewayRunner | `gateway/run.py` | 閘道器生命週期管理員；LRU 代理快取（128 個會話，TTL 1 小時） |
| PlatformAdapter | `gateway/platforms/base.py` | 所有閘道器平台介面卡必須實作的抽象基底類別 |
| delivery | `gateway/delivery.py` | 依平台分割訊息（Telegram：4096 字元，Discord：2000 字元） |
| SessionManager | `acp_adapter/session.py` | ACP 會話狀態、工具集擴充、模型切換、檢查點/回滾 |
| ACP Server | `acp_adapter/server.py` | 透過 ACP 暴露 AIAgent；以 ThreadPoolExecutor 處理同步呼叫 |
| CLI REPL | `cli.py` | prompt_toolkit 互動式迴圈、代理實例化、會話 DB 管理（3,000+ 行） |
| cmd_chat | `hermes_cli/main.py` | 頂層 CLI 入口；路由至 TUI 或 REPL，處理設定檔/憑證設定 |
| skill_commands | `agent/skill_commands.py` | CLI 與閘道器共用的 /斜線指令技能呼叫處理器 |
| Curator | `agent/curator.py` | 背景技能生命週期維護（獨立進程，輔助模型） |
| lazy_deps | `tools/lazy_deps.py` | 允許清單控管的執行期依賴安裝器 |
| hermes_state | `hermes_state.py` | 基於 ContextVar 的每任務家目錄作用域 |
| LSPClient | `agent/lsp/client.py` | 透過 stdin/stdout 的非同步 LSP 客戶端；每（語言，工作區）配對管理一個子伺服器進程 |
| chat_completion_helpers | `agent/chat_completion_helpers.py` | Anthropic 前綴快取標頭、串流、Token 計算 |
| web_search_registry | `agent/web_search_registry.py` | Web 搜尋提供者註冊與舊版備援鏈 |
| cron scheduler | `cron/scheduler.py` | 每 60 秒輪詢迴圈；啟動靜默模式代理；管理任務狀態與輸出日誌 |

---

## 資料流

### CLI 互動式會話

```
使用者輸入（終端機）
  └─> hermes_cli/main.py:cmd_chat()       # 設定檔/憑證檢查、環境設定
        └─> cli.py:main()                  # 載入 config.yaml + .env、建立 prompt_toolkit 應用程式
              └─> cli.py:_init_agent()     # 實例化 AIAgent、初始化 SessionDB
                    └─> agent/agent_init.py # 提供者偵測、上下文引擎初始化

使用者訊息提交
  └─> agent/conversation_loop.py:run_conversation()
        ├─ 系統提示詞組裝
        │    ├─ 從 SessionDB 恢復（前綴快取重用）或全新建置
        │    ├─ DEFAULT_AGENT_IDENTITY + SKILLS + MEMORY_GUIDANCE + 上下文檔案
        │    └─ 記憶體上下文區塊（MEMORY.md、USER.md、召回結果）
        ├─ agent/chat_completion_helpers.py:make_chat_completion()
        │    └─ 向 LLM 提供者發送串流請求（附 Anthropic 前綴快取標頭）
        ├─ 工具呼叫驗證 + 模糊比對修復（幻覺名稱）
        ├─ 執行前防護欄（檔案寫入 + 破壞性指令的檢查點）
        ├─ agent/tool_executor.py:execute_tool_calls_concurrent()
        │    └─ ThreadPoolExecutor（透過 _interrupt_requested 旗標傳播 Ctrl+C）
        ├─ 附加工具結果 → 壓縮檢查
        │    └─ context_compressor.py（如需要）：摘要中間輪次，建立新會話 ID
        └─ 回到模型呼叫迴圈 或 回傳最終回應

  └─> 會話持久化（SessionDB + JSON 日誌）
```

### 閘道器訊息會話

```
平台 Webhook（例如 Telegram 訊息）
  └─> gateway/platforms/telegram/_on_message()
        └─> gateway/run.py:GatewayRunner.on_message()
              └─ LRU 快取 AIAgent 查詢（最多 128 個，閒置 TTL 1 小時；找不到則建立新的）
                    └─> 與 CLI 相同的 run_conversation() 路徑
  └─> gateway/delivery.py:deliver_message()
        └─ 依平台分割回應 → 送至平台 API
```

### 排程任務（Cron）

```
cron/scheduler.py:tick()  [每 60 秒]
  └─> cron/jobs.get_due_jobs()
        └─> cron/scheduler.run_job()
              └─ 靜默模式 AIAgent（自動接受所有工具，無使用者提示）
                    └─> run_conversation() 路徑
              └─> gateway/delivery.py:deliver_message()
                    └─ 若訊息以 [SILENT] 為前綴則抑制傳遞
              └─> cron/jobs.advance_next_run()
              └─> 附加至 ~/.hermes/cron/outputs/{job_id}.log
```

### ACP（IDE 整合）會話

```
ACP 客戶端（例如 VS Code 擴充套件）
  └─> acp_adapter/__main__.py（ACP 伺服器）
        └─> acp_adapter/server.py（ThreadPoolExecutor）
              └─> acp_adapter/session.py（會話狀態、工具集、檢查點/回滾）
                    └─> AIAgent.run_conversation()（在執行緒中以同步方式呼叫）
```

---

## 入口點

| 場景 | 入口點 |
|------|--------|
| 互動式 CLI / TUI | `hermes_cli/main.py:cmd_chat()` → 路由至 `ui-tui/`（Node.js Ink）或 `cli.py:main()` |
| CLI REPL（直接） | `cli.py:main()` |
| 訊息閘道器常駐程序 | `gateway/run.py:GatewayRunner` |
| 排程 Cron 任務 | `cron/scheduler.py` |
| ACP IDE 整合 | `acp_adapter/__main__.py` |
| Web 儀表板 | `hermes_cli` 儀表板伺服器 → `web/` SPA |

---

## 外部整合

| 服務 / 協定 | 用途 | 模組 |
|------------|------|------|
| Anthropic API | 主要 LLM 提供者 | `agent/anthropic_adapter.py`、`plugins/model-providers/anthropic/` |
| OpenAI API + Codex OAuth | LLM 提供者（同時作為 OpenAI 相容代理目標） | `plugins/model-providers/openai-codex/` |
| AWS Bedrock | 透過 boto3 的 LLM 提供者 | `agent/bedrock_adapter.py`、`plugins/model-providers/bedrock/` |
| Azure AI Foundry | 具備 Entra ID / 受管理身分驗證的 LLM 提供者 | `agent/azure_identity_adapter.py` |
| Google Gemini | LLM 提供者（原生介面卡） | `agent/gemini_native_adapter.py` |
| DeepSeek、Kimi、MiniMax、Qwen、XAI、NVIDIA、HuggingFace、Arcee、Novita 等 | 額外 LLM 提供者 | `plugins/model-providers/` |
| Telegram | 訊息閘道器 | `gateway/platforms/telegram/`（python-telegram-bot） |
| Discord | 訊息閘道器 | `gateway/platforms/discord/`（discord.py） |
| Slack | 訊息閘道器 | `gateway/platforms/slack/`（slack-bolt + slack-sdk） |
| Matrix | 訊息閘道器 | `gateway/platforms/matrix/`（mautrix） |
| WhatsApp、Signal、WeChat、飛書、釘釘、IRC、Line、Teams、SMS、SimplexChat、元寶 | 訊息閘道器 | `gateway/platforms/` |
| Exa、Firecrawl、Parallel Web、DuckDuckGo、Brave Search、SearXNG、Tavily | Web 搜尋後端 | `agent/web_search_registry.py`、`plugins/web/` |
| Honcho、Hindsight、Mem0、Supermemory | 外部記憶體提供者 | `plugins/memory/` |
| ElevenLabs TTS、Azure Edge TTS | 文字轉語音 | `plugins/` |
| Faster-Whisper | 語音轉文字 | `plugins/` |
| Fal.ai | 圖片生成 | `plugins/image-gen/` |
| Modal、Daytona、Vercel | 工具執行用無伺服器運算 | `tools/` |
| Browserbase | 瀏覽器自動化工具的雲端瀏覽器（CDP） | `tools/` |
| Google Workspace APIs | 雲端硬碟、日曆、Gmail 整合 | `tools/` / MCP 工具 |
| Microsoft Graph APIs | Microsoft 365 整合 | `tools/` |
| ACP v0.9.0 | IDE/編輯器整合協定 | `acp_adapter/` |
| MCP v1.26.0 | 工具擴充協定 | `tools/mcp_tool.py` |
| GitHub JWT | 技能登錄表的 Skills Hub 機器人身分認證 | `acp_registry/` |

---

## 注意事項

- **精確版本鎖定是有意為之，而非疏漏。** `pyproject.toml` 中所有依賴均使用 `==X.Y.Z` 鎖定，這是繼 2026-05-12 Mini Shai-Hulud 蠕蟲事件（mistralai 2.4.6）後刻意採用的供應鏈強化措施。新版本只有透過明確的版本升級與 `uv lock` 重新生成才能到達使用者端。

- **`run_agent.py` 是外觀，而非真實邏輯所在。** 其中的 `AIAgent` 類別很薄——真正的初始化在 `agent/agent_init.py`（約 1,400 行），對話驅動在 `agent/conversation_loop.py`（約 3,900 行）。除錯代理迴圈就要讀 `conversation_loop.py`。

- **上下文壓縮是破壞性且不可逆的。** 當 `ContextCompressor` 觸發時，原始對話輪次會被摘要取代。系統會建立新的會話 ID（透過 `parent_session_id` 連結），舊的可在歷史中找到但無法恢復。

- **工具註冊表使用 AST 掃描，而非直接引入。** `tools/registry.py` 透過靜態掃描 Python AST 來探索工具註冊，避免了在啟動時引入所有工具模組所產生的循環引入問題。工具必須在模組層級呼叫 `registry.register()` 才能被探索到。

- **`model_tools.py` 擁有單一長駐的 asyncio 事件迴圈。** 它不使用 `asyncio.run()` 逐次呼叫。這是有意為之——重用事件迴圈可防止工具回呼從工作執行緒被呼叫時出現「Event loop is closed」錯誤。

- **外部提供者的記憶體上下文是非權威性的。** 外部提供者（Honcho、Mem0 等）返回的記憶體以「背景參考」形式注入。系統提示詞記憶體區塊（MEMORY.md、USER.md）才是權威來源。外部提供者的衝突值應視為提示而非指令。

- **延遲依賴安裝受允許清單控管。** `tools/lazy_deps.py` 只會安裝其明確允許清單上的套件。在設定中設置 `security.allow_lazy_installs: false` 可完全停用執行期安裝。這是安全控制機制，而非單純的便利功能。

- **Anthropic 自適應思考模型（4.6+）拒絕 `temperature`/`top_p`/`top_k` 參數。** 傳遞這些參數會導致 400 錯誤。介面卡必須偵測思考模型並在請求前移除這些參數。

- **SQLite 使用 WAL 模式但有備援機制。** `state.db` 以 WAL 模式運行以獲得效能優勢，但在不支援 WAL 的 NFS/SMB/WSL1 掛載上會自動退回 journal 模式。假設 WAL 特定行為（例如並發讀取者）的程式碼在這些檔案系統上會出錯。

- **設定檔旗標必須在任何模組引入前解析。** `hermes_cli/main.py` 中的 `--profile/-p` 設定啟用的提供者設定檔。如果模組引入先發生，就會載入錯誤的提供者設定。這透過 `cmd_chat()` 中的參數解析順序強制執行。

- **`hermes_state.py` 使用 `ContextVar` 進行家目錄作用域隔離。** 這是多租戶 cron/閘道器執行時避免各 `~/.hermes` 狀態互相污染的方式。直接使用 `Path.home()` 解析路徑的程式碼會破壞此隔離機制。

- **閘道器代理快取有上限。** `GatewayRunner` 最多在 LRU 快取中保留 128 個並發代理會話，閒置 TTL 為 1 小時。超過 1 小時回來的使用者會得到全新代理（無記憶體內對話狀態），但會話資料庫仍持久化在磁碟上。

- **Windows 支援為早期測試版；WSL2 是經過驗證的路徑。** 原生 Windows 路徑與子進程處理有專屬的因應方案（例如 `tzdata` 套件、隱藏主控台視窗）。WSL2 是 Windows 使用者的推薦環境。

- **技能（Skills）與工具（Tools）在架構上是截然不同的。** 技能是透過斜線指令呼叫的 Markdown 檔案加 shell/API 包裝器。工具是在工具註冊表中登錄、可由 LLM 在對話輪次中呼叫的 Python 模組。混淆兩者會導致把技能內容放進 `tools/` 或把 Python 邏輯放進 `skills/`，兩者均屬錯誤。

- **推理標籤在顯示時被移除但保留於資料庫中。** `<think>`、`<REASONING_SCRATCHPAD>` 及類似標籤在使用者看到的內容中被剝除，但原封不動地儲存在 SessionDB 中。查詢資料庫會看到它們；渲染的 UI 則不會顯示。
