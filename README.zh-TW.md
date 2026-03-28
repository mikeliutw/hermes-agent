<p align="center">
  <img src="assets/banner.png" alt="Hermes Agent" width="100%">
</p>

# Hermes Agent ☤

<p align="center">
  <a href="https://hermes-agent.nousresearch.com/docs/"><img src="https://img.shields.io/badge/文件-hermes--agent.nousresearch.com-FFD700?style=for-the-badge" alt="文件"></a>
  <a href="https://discord.gg/NousResearch"><img src="https://img.shields.io/badge/Discord-5865F2?style=for-the-badge&logo=discord&logoColor=white" alt="Discord"></a>
  <a href="https://github.com/NousResearch/hermes-agent/blob/main/LICENSE"><img src="https://img.shields.io/badge/授權-MIT-green?style=for-the-badge" alt="授權: MIT"></a>
  <a href="https://nousresearch.com"><img src="https://img.shields.io/badge/開發者-Nous%20Research-blueviolet?style=for-the-badge" alt="由 Nous Research 開發"></a>
</p>

**由 [Nous Research](https://nousresearch.com) 打造的自我進化 AI Agent。** 這是唯一內建學習迴圈的 Agent — 它能從使用經驗中建立技能、在使用過程中持續改進技能、主動記錄知識、搜尋自己過去的對話紀錄，並跨對話建立對使用者的深層理解。可以部署在 $5 的 VPS、GPU 叢集或幾乎不需要成本的無伺服器架構上。不受限於本機電腦 — 你可以透過 Telegram 與它對話，同時讓它在雲端 VM 上運行工作。

可以使用任意模型 — [Nous Portal](https://portal.nousresearch.com)、[OpenRouter](https://openrouter.ai)（200+ 模型）、[z.ai/GLM](https://z.ai)、[Kimi/Moonshot](https://platform.moonshot.ai)、[MiniMax](https://www.minimax.io)、OpenAI 或自己的端點。使用 `hermes model` 即可切換，無需修改程式碼，不受平台綁定。

<table>
<tr><td><b>真正的終端介面</b></td><td>完整 TUI，支援多行編輯、斜線指令自動補全、對話歷史、中斷並重導向，以及串流工具輸出。</td></tr>
<tr><td><b>跟著你走</b></td><td>Telegram、Discord、Slack、WhatsApp、Signal 和 CLI — 全部由單一 gateway 進程驅動。支援語音備忘錄轉錄、跨平台對話連續性。</td></tr>
<tr><td><b>閉環學習</b></td><td>Agent 管理的記憶系統，定期主動提示。複雜任務完成後自動建立技能，技能在使用中自我改進。FTS5 對話搜尋搭配 LLM 摘要，實現跨對話記憶召回。支援 <a href="https://github.com/plastic-labs/honcho">Honcho</a> 使用者建模，相容 <a href="https://agentskills.io">agentskills.io</a> 開放標準。</td></tr>
<tr><td><b>排程自動化</b></td><td>內建 cron 排程器，可推送至任意平台。用自然語言設定每日報告、每夜備份、每週審查 — 全部無人值守自動執行。</td></tr>
<tr><td><b>委派與並行</b></td><td>可生成隔離子 Agent 進行並行工作流。也可撰寫 Python 腳本透過 RPC 呼叫工具，將多步驟流程壓縮為零上下文成本的單一回合。</td></tr>
<tr><td><b>隨處部署</b></td><td>六種終端後端 — 本機、Docker、SSH、Daytona、Singularity 和 Modal。Daytona 和 Modal 提供無伺服器持久化 — 閒置時 Agent 環境自動休眠、按需喚醒，閒置期間幾乎零成本。可在 $5 VPS 或 GPU 叢集上執行。</td></tr>
<tr><td><b>研究就緒</b></td><td>批次軌跡生成、Atropos RL 環境、軌跡壓縮，用於訓練下一代工具呼叫模型。</td></tr>
</table>

---

## 快速安裝

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

支援 Linux、macOS 和 WSL2。安裝程式自動處理所有事項 — Python、Node.js、相依套件和 `hermes` 指令。除 git 外無任何前置需求。

> **Windows：** 不支援原生 Windows。請先安裝 [WSL2](https://learn.microsoft.com/zh-tw/windows/wsl/install)，再執行上方指令。

安裝完成後：

```bash
source ~/.bashrc    # 重新載入 shell（或：source ~/.zshrc）
hermes              # 開始對話！
```

---

## 入門指南

```bash
hermes              # 互動式 CLI — 開始對話
hermes model        # 選擇 LLM 提供商和模型
hermes tools        # 設定要啟用的工具
hermes config set   # 設定個別設定值
hermes gateway      # 啟動訊息閘道（Telegram、Discord 等）
hermes setup        # 執行完整設定精靈（一次設定所有內容）
hermes claw migrate # 從 OpenClaw 遷移（如果你是從 OpenClaw 來的）
hermes update       # 更新至最新版本
hermes doctor       # 診斷任何問題
```

📖 **[完整文件 →](https://hermes-agent.nousresearch.com/docs/)**

---

## CLI 與訊息平台快速對照

Hermes 有兩個入口：用 `hermes` 啟動終端 UI，或執行 gateway 並透過 Telegram、Discord、Slack、WhatsApp、Signal 或 Email 與它互動。進入對話後，許多斜線指令在兩種介面上通用。

| 操作 | CLI | 訊息平台 |
|------|-----|----------|
| 開始對話 | `hermes` | 執行 `hermes gateway setup` + `hermes gateway start`，然後向 bot 傳送訊息 |
| 開始新對話 | `/new` 或 `/reset` | `/new` 或 `/reset` |
| 切換模型 | `/model [provider:model]` | `/model [provider:model]` |
| 設定人格 | `/personality [name]` | `/personality [name]` |
| 重試或撤銷最後一輪 | `/retry`、`/undo` | `/retry`、`/undo` |
| 壓縮上下文 / 查看使用量 | `/compress`、`/usage`、`/insights [--days N]` | `/compress`、`/usage`、`/insights [days]` |
| 瀏覽技能 | `/skills` 或 `/<skill-name>` | `/skills` 或 `/<skill-name>` |
| 中斷當前工作 | `Ctrl+C` 或傳送新訊息 | `/stop` 或傳送新訊息 |
| 平台特定狀態 | `/platforms` | `/status`、`/sethome` |

完整指令列表請參閱 [CLI 指南](https://hermes-agent.nousresearch.com/docs/user-guide/cli) 和 [訊息閘道指南](https://hermes-agent.nousresearch.com/docs/user-guide/messaging)。

---

## 文件索引

所有文件均在 **[hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)**：

| 章節 | 內容 |
|------|------|
| [快速入門](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart) | 安裝 → 設定 → 2 分鐘內完成第一次對話 |
| [CLI 使用](https://hermes-agent.nousresearch.com/docs/user-guide/cli) | 指令、快捷鍵、人格、對話記錄 |
| [設定](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) | 設定檔、提供商、模型、所有選項 |
| [訊息閘道](https://hermes-agent.nousresearch.com/docs/user-guide/messaging) | Telegram、Discord、Slack、WhatsApp、Signal、Home Assistant |
| [安全性](https://hermes-agent.nousresearch.com/docs/user-guide/security) | 指令審批、DM 配對、容器隔離 |
| [工具與工具集](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) | 40+ 種工具、工具集系統、終端後端 |
| [技能系統](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) | 程序記憶、Skills Hub、建立技能 |
| [記憶](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) | 持久記憶、使用者檔案、最佳實踐 |
| [MCP 整合](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) | 連接任意 MCP 伺服器以擴充功能 |
| [Cron 排程](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) | 排程任務與平台推送 |
| [情境檔案](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) | 影響每次對話的專案情境 |
| [架構](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) | 專案結構、Agent 迴圈、關鍵類別 |
| [貢獻指南](https://hermes-agent.nousresearch.com/docs/developer-guide/contributing) | 開發環境設定、PR 流程、程式碼風格 |
| [CLI 參考](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) | 所有指令與旗標 |
| [環境變數](https://hermes-agent.nousresearch.com/docs/reference/environment-variables) | 完整環境變數參考 |

---

## 從 OpenClaw 遷移

如果你是從 OpenClaw 來的，Hermes 可以自動匯入你的設定、記憶、技能和 API 金鑰。

**首次設定時：** 設定精靈（`hermes setup`）會自動偵測 `~/.openclaw`，並在開始設定前提供遷移選項。

**安裝後任何時候：**

```bash
hermes claw migrate              # 互動式遷移（完整預設）
hermes claw migrate --dry-run    # 預覽將會遷移的內容
hermes claw migrate --preset user-data   # 不遷移密鑰的遷移
hermes claw migrate --overwrite  # 覆蓋現有衝突
```

遷移內容：
- **SOUL.md** — 人格檔案
- **記憶** — MEMORY.md 和 USER.md 條目
- **技能** — 使用者建立的技能 → `~/.hermes/skills/openclaw-imports/`
- **指令白名單** — 審批模式
- **訊息設定** — 平台設定、允許用戶、工作目錄
- **API 金鑰** — 白名單密鑰（Telegram、OpenRouter、OpenAI、Anthropic、ElevenLabs）
- **TTS 資源** — 工作區音訊檔案
- **工作區指示** — AGENTS.md（搭配 `--workspace-target`）

詳見 `hermes claw migrate --help`，或使用 `openclaw-migration` 技能進行互動式的 Agent 引導遷移（支援 dry-run 預覽）。

---

## 貢獻

歡迎貢獻！請參閱[貢獻指南](https://hermes-agent.nousresearch.com/docs/developer-guide/contributing)了解開發環境設定、程式碼風格和 PR 流程。

貢獻者快速入門：

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"
python -m pytest tests/ -q
```

> **RL 訓練（選用）：** 若要開發 RL/Tinker-Atropos 整合：
> ```bash
> git submodule update --init tinker-atropos
> uv pip install -e "./tinker-atropos"
> ```

---

## 社群

- 💬 [Discord](https://discord.gg/NousResearch)
- 📚 [Skills Hub](https://agentskills.io)
- 🐛 [問題回報](https://github.com/NousResearch/hermes-agent/issues)
- 💡 [討論區](https://github.com/NousResearch/hermes-agent/discussions)

---

## 授權

MIT — 詳見 [LICENSE](LICENSE)。

由 [Nous Research](https://nousresearch.com) 開發。
