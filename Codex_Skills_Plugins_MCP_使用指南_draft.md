# Codex 擴充功能入門：Skills / Plugins / MCP（Draft v0.1）

> 適用：OpenAI Codex CLI（本機驗證版本 codex-cli 0.142.0，Windows）與 Codex 桌面 App / VS Code 擴充。
> 目的：讓同學看完就知道「這三個東西是什麼、差在哪、怎麼裝、怎麼用」。
> 狀態：草稿，待審閱。

---

## 0. 三分鐘看懂：Skill、Plugin、MCP 差在哪？

| | **Skill** | **MCP Server** | **Plugin** |
|---|---|---|---|
| 一句話 | 教 Codex「怎麼做某件事」的說明書 | 給 Codex「新工具 / 新資料來源」 | 把 Skill + MCP + 設定「打包成一個安裝包」 |
| 本質 | 一個資料夾，裡面有 `SKILL.md`（Markdown 指令） | 一個獨立程式（本機 stdio 或遠端 HTTP），透過 MCP 協定提供工具 | 一個含 `.codex-plugin/plugin.json` 的資料夾 |
| 改變了什麼 | 模型的**行為 / 流程 / 知識** | 模型**能碰到的外部系統**（GitHub、Slack、DB、瀏覽器…） | 一次把上述兩者裝好 |
| 生活比喻 | SOP 手冊 | 新的外接設備（印表機、掃描器） | 整套「開箱即用」的套件組 |
| 怎麼叫用 | 打 `$skill-name`，或 Codex 依描述自動選用 | Codex 依需要自動呼叫工具；`/mcp` 查看狀態 | 裝完重新開 session 即生效 |
| 存放位置 | `~/.codex/skills/` 或 `~/.agents/skills/`（專案：`.agents/skills/`） | `~/.codex/config.toml` 的 `[mcp_servers.xxx]` | `~/.codex/plugins/` |

**記憶口訣**：Skill = 教它怎麼做；MCP = 給它新工具；Plugin = 一鍵打包裝。

---

## 1. 前置準備

```bash
# 確認版本（skills / plugins 都是 2025 Q4 以後才有，建議更新到最新）
codex --version

# 更新到最新
codex update

# 檢查安裝、登入、設定是否健康
codex doctor
```

重要路徑（Windows 上 `~` = `C:\Users\<你的帳號>`）：

| 路徑 | 用途 |
|---|---|
| `~/.codex/config.toml` | 全域設定（MCP、model、trust 專案等） |
| `~/.codex/skills/` | 個人 skills（本機 0.142.0 執行檔內有此路徑） |
| `~/.agents/skills/` | 官方文件目前寫的個人 skills 路徑（跨 agent 共用） |
| `<專案>/.agents/skills/` | 專案級 skills，會跟著 repo 走 |
| `~/.codex/plugins/` | 已安裝的 plugins |
| `<專案>/.codex/config.toml` | 專案級設定（需為 trusted 專案） |

> 兩個 skills 路徑都被支援，擇一即可。不確定時，裝完在 Codex 內打 `/skills` 看有沒有列出來。

---

## 2. Skills（技能）

### 2.1 長什麼樣

```
my-skill/
├── SKILL.md            # 必要：frontmatter + 指令
├── scripts/            # 可選：可執行腳本
├── references/         # 可選：參考文件
├── assets/             # 可選：範本、素材
└── agents/openai.yaml  # 可選：顯示名稱、icon、是否允許自動觸發
```

`SKILL.md` 最小範例：

```markdown
---
name: my-skill
description: 什麼情況該用、什麼情況不該用（這行是 Codex 決定要不要自動啟用的主要依據）
---

接下來是給 Codex 看的指令、步驟、檢查清單……
```

### 2.2 怎麼使用

| 方式 | 做法 |
|---|---|
| 明確指定 | 在 Codex 對話框打 `$` 會跳出清單，例如 `$skill-creator 幫我做一個…` |
| 列出全部 | 打 `/skills` |
| 自動觸發 | 你描述的任務符合某個 skill 的 `description`，Codex 會自己套用 |

### 2.3 安裝方式（四選一）

**A. 手動複製（最通用，任何來源都行）**

```bash
# 把整個 skill 資料夾放進去，資料夾內要有 SKILL.md
mkdir -p ~/.codex/skills
cp -r <下載的 skill 資料夾> ~/.codex/skills/
# 重新開一個 Codex session 即生效
```

**B. `$skill-installer`（Codex 內建，從 OpenAI 目錄裝）**

在 Codex 對話框直接打：

```
$skill-installer gh-fix-ci
$skill-installer install https://github.com/<owner>/<repo>/tree/main/<path-to-skill>
```

**C. `npx skills add`（Vercel 的跨 agent 安裝器，支援 GitHub 任意 repo）**

```bash
# 裝整個 repo 的所有 skills 到 Codex（全域）
npx skills add anthropics/skills -a codex -g

# 只裝其中一個 skill
npx skills add anthropics/skills --skill mcp-builder -a codex -g

# 管理
npx skills list
npx skills remove <skill-name>
npx skills find <關鍵字>
```

**D. 透過 Plugin 安裝**（見第 3 節，例如 superpowers 就是用這種方式）

### 2.4 停用某個 skill

在 `~/.codex/config.toml` 加上：

```toml
[[skills.config]]
path = "C:/Users/<你>/.codex/skills/<skill-name>/SKILL.md"
enabled = false
```

### 2.5 自己做一個 skill

在 Codex 打 `$skill-creator`，它會互動式幫你產生資料夾與 `SKILL.md`。
做完放到 `~/.codex/skills/`，重開 session 用 `/skills` 確認。

---

## 3. Plugins（外掛套件）

### 3.1 是什麼

一個 plugin 可以同時包含：**skills、MCP servers、App 連接器（如 Google Drive、Slack）、hooks**。
裝一次，全部到位。OpenAI 官方維護一個 marketplace 叫 `openai-curated`，Codex 預設就有。

### 3.2 安裝（TUI 內最簡單）

```
/plugins            ← 打開 plugin 瀏覽器
搜尋 superpowers    ← 選到後按 Install Plugin
（裝完重新開一個 session）
```

- 按 **Space** 可以在不解除安裝的情況下暫時停用 / 啟用。
- 解除安裝只移除 plugin 本身，另外連上的 MCP server 要自己手動移除。

### 3.3 安裝（命令列）

```bash
# 看目前有哪些 marketplace、有哪些 plugin
codex plugin marketplace list
codex plugin list

# 從官方 marketplace 裝
codex plugin add superpowers@openai-curated

# 加入第三方 marketplace（支援 owner/repo、Git URL、本機路徑）
codex plugin marketplace add https://github.com/ComposioHQ/composio-plugin-openai.git
codex plugin add composio@composio

# 更新 / 移除
codex plugin marketplace upgrade
codex plugin remove <plugin-name>
```

### 3.4 本機 openai-curated marketplace 目前提供的 plugins（2026-09 實測）

linear、atlassian-rovo、google-calendar、gmail、slack、teams、sharepoint、outlook-email、outlook-calendar、canva、figma、stripe、vercel、game-studio、**superpowers**、github、circleci、google-drive、notion、cloudflare、sentry、build-ios-apps、build-macos-apps、build-web-apps、build-web-data-visualization、test-android-apps

### 3.5 Plugin 資料夾結構（想自己做時參考）

```
my-plugin/
├── .codex-plugin/plugin.json   # 必要：manifest
├── skills/                     # 內含的 skills
├── .mcp.json                   # 內含的 MCP servers
├── .app.json                   # App 連接器
├── hooks.json
└── assets/
```

範例 repo：https://github.com/openai/plugins

---

## 4. MCP（Model Context Protocol）

### 4.1 是什麼

MCP 是一個開放協定，讓 Codex 能呼叫外部「工具」（tools）與讀取「資源」。
MCP server 有兩種：

| 類型 | 說明 | 例子 |
|---|---|---|
| **stdio** | 本機啟動一支程式，用標準輸入輸出溝通 | `npx -y @upstash/context7-mcp` |
| **Streamable HTTP** | 遠端服務，用 URL 連 | `https://mcp.example.com/mcp` |

### 4.2 加入 MCP server（命令列，推薦）

```bash
# stdio：-- 後面接啟動指令
codex mcp add context7 -- npx -y @upstash/context7-mcp

# 帶環境變數（要放在 -- 之前）
codex mcp add github --env GITHUB_TOKEN=ghp_xxx -- npx -y @modelcontextprotocol/server-github

# 遠端 HTTP
codex mcp add example --url https://mcp.example.com/mcp

# 遠端 + bearer token（從環境變數讀）
codex mcp add example --url https://mcp.example.com/mcp --bearer-token-env-var MY_TOKEN

# 管理
codex mcp list
codex mcp get <name>
codex mcp remove <name>
codex mcp login <name>     # OAuth 登入
codex mcp logout <name>
```

### 4.3 直接改 `config.toml`

```toml
# stdio
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]
startup_timeout_sec = 10
tool_timeout_sec = 60

[mcp_servers.context7.env]
MY_ENV_VAR = "value"

# 遠端 HTTP
[mcp_servers.remote-example]
url = "https://mcp.example.com/mcp"
bearer_token_env_var = "MY_TOKEN"     # 或 auth = "oauth"

# 常用選項（兩種類型皆可）
# enabled = true / false
# enabled_tools = ["read", "search"]       # 白名單
# disabled_tools = ["delete"]              # 黑名單
# default_tools_approval_mode = "prompt"   # auto / prompt / writes / approve
```

> 注意：頂層 key 是 `mcp_servers`（底線），不是 Claude / Cursor 用的 `mcpServers`。

### 4.4 在 Codex 內查看

打 `/mcp` 可看到目前連上的 server 與工具。桌面 App / VS Code：Settings → MCP servers → Add server。

### 4.5 專案級 MCP

在專案根目錄建 `.codex/config.toml`，同樣寫 `[mcp_servers.xxx]`，只對該專案生效（專案需為 trusted）。

---

## 5. 推薦 Skills 清單

### 5.1 你已知的 9 個

| # | Skill | 用途 | 來源 | Codex 安裝方式 | 備註 |
|---|---|---|---|---|---|
| 1 | **Superpowers** | 一整套開發方法論：brainstorming → 寫計畫 → TDD → 系統化除錯 → code review → 收尾分支。裝了之後 Codex 做事會「先想再動手」 | obra/superpowers | `/plugins` 搜 superpowers → Install，或 `codex plugin add superpowers@openai-curated` | 官方 marketplace 直接有，最推薦第一個裝 |
| 2 | **artifact-design** | （Claude Code 內建）產生互動式網頁 draft / 設計指引 | Anthropic（Claude Code 專屬） | 無法直接裝到 Codex | Codex 的對應替代：anthropics/skills 的 `web-artifacts-builder` + `frontend-design` |
| 3 | **Academic Research Skills** | 學術研究流程：文獻回顧、寫論文、同儕審查、修改、定稿 | Imbad0202/academic-research-skills（4 個 skill、10 階段）；另有 ngtiendong/Academic-Research-Agent-Skill（偏碩博生研究流程） | `npx skills add Imbad0202/academic-research-skills -a codex -g`，或 clone 後複製到 `~/.codex/skills/` | 原設計給 Claude Code，slash command 在 Codex 需改用 `$` 或直接描述 |
| 4 | **Karpathy Guidelines** | 四條寫程式行為準則：先想再寫、簡單優先、外科手術式修改、目標導向可驗證 | swarmclawai/andrej-karpathy-skills（有 Codex 版） | `npx @swarmclawai/andrej-karpathy-skills --agent codex --dest <專案路徑>`（會寫進專案 `AGENTS.md`） | 也可把 `skills/karpathy-guidelines` 資料夾複製到 `~/.codex/skills/` |
| 5 | **Frontend Design** | 做出「不像 AI 樣板」的前端 UI，含字型、配色、版面原則 | anthropics/skills | `npx skills add anthropics/skills --skill frontend-design -a codex -g` | Anthropic 官方，安裝數最高的 skill |
| 6 | **Webapp Testing** | 用 Playwright 對本機網站做測試、截圖、看 console log | anthropics/skills | `npx skills add anthropics/skills --skill webapp-testing -a codex -g` | 需 `pip install playwright` |
| 7 | **MCP Builder** | 帶你寫一個品質良好的 MCP server（Python / TypeScript） | anthropics/skills | `npx skills add anthropics/skills --skill mcp-builder -a codex -g` | 做完的 server 用 `codex mcp add` 接上 |
| 8 | **Skill Creator** | 互動式建立 / 優化 skill | Codex 內建（`$skill-creator`）；anthropics/skills 也有一份 | 內建版免安裝；Anthropic 版 `npx skills add anthropics/skills --skill skill-creator -a codex -g` | 先用內建版即可 |
| 9 | **Composio** | 讓 Codex 透過 Composio CLI 操作 1500+ SaaS（GitHub、Gmail、Notion、Jira…），免自己接 OAuth | ComposioHQ | 1) `curl -fsSL https://composio.dev/install \| sh`  2) `composio login`  3) `composio setup --target codex`（或 `codex plugin marketplace add https://github.com/ComposioHQ/composio-plugin-openai.git` + `codex plugin add composio@composio`） | 需要 Composio 帳號；Windows 建議在 WSL 或 Git Bash 執行安裝腳本 |

### 5.2 網路上另外常被推薦的（補充）

| Skill | 用途 | 來源 | 安裝 |
|---|---|---|---|
| **gh-fix-ci** | 診斷並修 GitHub Actions 失敗 | openai/skills（官方 curated） | `$skill-installer gh-fix-ci` |
| **gh-address-comments** | 逐條處理 PR review 留言 | openai/skills | `$skill-installer gh-address-comments` |
| **yeet** | 一鍵 stage → commit → push → 開 PR | openai/skills | `$skill-installer yeet` |
| **playwright** / **playwright-interactive** | 瀏覽器自動化 / 互動除錯 | openai/skills | `$skill-installer playwright` |
| **security-best-practices** / **security-threat-model** | 安全檢查與威脅建模 | openai/skills | `$skill-installer security-best-practices` |
| **cli-creator** | 做出輸出穩定 JSON 的 CLI 工具 | openai/skills | `$skill-installer cli-creator` |
| **migrate-to-codex** | 把 Claude Code 的設定（CLAUDE.md、skills）轉成 Codex 格式 | openai/skills | `$skill-installer migrate-to-codex` |
| **create-plan** | 先產生實作計畫再動手 | openai/skills（experimental） | `$skill-installer install https://github.com/openai/skills/tree/main/skills/.experimental/create-plan` |
| **grill-me** | 用蘇格拉底式提問把你的計畫問到沒有漏洞 | mattpocock/skills | `npx skills add mattpocock/skills --skill grill-me -a codex -g` |
| **handoff** | 把目前 session 壓縮成 markdown，交接給下一個 session / 同事 | mattpocock/skills | `npx skills add mattpocock/skills --skill handoff -a codex -g` |
| **docx / pptx / xlsx / pdf** | 產生與編輯 Office 文件、PDF | anthropics/skills | `npx skills add anthropics/skills --skill docx -a codex -g`（其餘同理） |
| **scientific-agent-skills** | 165 個科學研究 skills（生物、化學、醫學…） | K-Dense-AI/scientific-agent-skills | `npx skills add K-Dense-AI/scientific-agent-skills -a codex -g` |

> openai/skills 這個 repo 已標示 deprecated（改推 openai/plugins），但 `$skill-installer` 仍可從它安裝，且 curated 清單仍在。

### 5.3 建議「入門套餐」

第一週先裝這四個就夠：

1. **superpowers**（方法論）→ `/plugins` 搜尋安裝
2. **karpathy-guidelines**（行為準則）→ 寫進專案 `AGENTS.md`
3. **frontend-design**（前端 UI）
4. **gh-fix-ci** 或 **yeet**（Git / CI 日常）

MCP 方面先加一個 **context7**（即時查函式庫文件）：

```bash
codex mcp add context7 -- npx -y @upstash/context7-mcp
```

---

## 6. 常見問題與注意事項

| 狀況 | 處理 |
|---|---|
| 裝了但 `/skills` 看不到 | 重新開一個 session；確認資料夾內有 `SKILL.md` 且 frontmatter 有 `name` / `description` |
| 專案級設定沒生效 | 專案要先被 trust（第一次開專案時選 trusted，或在 `config.toml` 的 `[projects.'<路徑>']` 設 `trust_level = "trusted"`） |
| MCP server 起不來 | `codex mcp list` 看狀態；調大 `startup_timeout_sec`；Windows 常見是 `npx` 路徑問題，可改用 `cmd /c npx ...` |
| Windows 路徑寫法 | `config.toml` 裡用正斜線 `C:/Users/...` 或雙反斜線 |
| skill 之間互相衝突 | 用 `[[skills.config]] enabled = false` 關掉其中一個 |
| 安全 | 不要把 token 寫進 `SKILL.md`；MCP token 用 `bearer_token_env_var` 或 `env` 讀環境變數；第三方 skill 裝之前先看一眼 `SKILL.md` 與 `scripts/` |
| 同一份 skill 想給 Claude Code 也用 | `SKILL.md` 格式是通用標準，同一資料夾複製到 `~/.claude/skills/` 即可；或用 `npx skills add ... -a codex -a claude-code` 一次裝兩邊 |

---

## 7. 參考資料

官方
- Codex Skills 文件：https://developers.openai.com/codex/skills
- Codex Plugins 文件：https://developers.openai.com/codex/plugins
- Codex MCP 文件：https://developers.openai.com/codex/mcp
- Codex 設定檔：https://developers.openai.com/codex/config-basic
- OpenAI plugins 範例 repo：https://github.com/openai/plugins
- OpenAI skills 目錄（deprecated 但仍可用）：https://github.com/openai/skills

Skills 來源
- Superpowers：https://github.com/obra/superpowers
- Anthropic 官方 skills：https://github.com/anthropics/skills
- Karpathy Guidelines：https://github.com/swarmclawai/andrej-karpathy-skills
- Academic Research Skills：https://github.com/imbad0202/academic-research-skills 、 https://github.com/ngtiendong/Academic-Research-Agent-Skill
- Composio：https://docs.composio.dev/docs/agent-plugins 、 https://github.com/composio-community/awesome-codex-skills
- mattpocock/skills：https://github.com/mattpocock/skills
- Vercel `npx skills` 安裝器：https://github.com/vercel-labs/skills 、 https://skills.sh

整理文章
- Firecrawl「Best Codex Skills」：https://www.firecrawl.dev/blog/best-codex-skills
- Codex Knowledge Base（plugin marketplace 說明）：https://codex.danielvaughan.com/2026/05/08/codex-cli-plugin-marketplace-remote-install-workspace-sharing-bundled-hooks/
- Simon Willison「OpenAI are quietly adopting skills」：https://simonw.substack.com/p/openai-are-quietly-adopting-skills
- awesome-codex-plugins：https://github.com/hashgraph-online/awesome-codex-plugins

---

## 待你確認的問題（審閱用）

1. 「artifact-design」你指的是 Claude Code 這邊的那個嗎？Codex 沒有對應內建，我先寫替代方案，要不要保留這一列？
2. Academic Research Skills 你用的是哪一個 repo？我列了兩個最常見的。
3. 同學的程度：要不要再加一段「第一次跑 Codex」（登入、trust 專案、approval mode）？
4. 要不要做成投影片或網頁版（可分享連結）？
