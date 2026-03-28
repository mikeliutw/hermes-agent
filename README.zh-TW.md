# Hermes Agent 專案概覽（中文）

Hermes Agent 是由 Nous Research 打造的自我學習 AI 代理。它在對話過程中會建立與改進技能、保存長期記憶、搜尋歷史對話，並能在多種平台（終端機與訊息平台）上運行，同時支援多家模型供應商與自訂端點。

## 核心特性
- **自我學習迴圈**：自動從任務產生技能並在使用中優化，定期提示代理保存重要記憶。
- **多平台介面**：內建 TUI 終端體驗與統一的訊息閘道，支援 Telegram、Discord、Slack、WhatsApp、Signal 等。
- **豐富工具組**：40+ 工具與多種終端後端（本機、Docker、SSH、Daytona、Singularity、Modal），可啟動子代理並平行化工作流。
- **記憶與技能**：FTS5 會話搜尋、使用者模型、Skills Hub 瀏覽/安裝，以及符合 agentskills.io 標準的技能檔。
- **排程自動化**：內建 cron，能將結果送往任一平台（每日報告、備份、審計等）。
- **部署彈性**：可在 $5 VPS、GPU 叢集或近乎零閒置成本的 serverless 後端上運行，不綁定本機。

## 快速開始
```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc   # 或 source ~/.zshrc
hermes             # 啟動對話
```
> Windows 需使用 WSL2。

常用指令：
- `hermes model`：選擇模型供應商與型號（OpenRouter、Nous Portal、OpenAI、Moonshot 等）。
- `hermes tools`：開關工具組。
- `hermes setup`：首次安裝時的互動式設定精靈。
- `hermes gateway`：啟動訊息平台閘道。
- `hermes doctor`：診斷常見問題。
- `hermes update`：更新至最新版本。

## 架構一瞥
- `run_agent.py`：AIAgent 主迴圈與工具呼叫流程。
- `model_tools.py` / `toolsets.py`：工具發現、註冊與工具組定義。
- `hermes_cli/`：CLI 入口、命令註冊、設定載入與主題系統。
- `tools/`：各工具實作（終端、檔案、網路、瀏覽器、自動化等）。
- `gateway/`：統一的訊息平台適配器與會話存儲。
- `agent/`：提示組裝、上下文壓縮、模型中繼資料、展示元件、技能命令。
- `tests/`：Pytest 套件，覆蓋 CLI、工具、閘道與工具組解析。

## 文件與社群
- 官方文件（英文）：https://hermes-agent.nousresearch.com/docs/
- Discord 社群：https://discord.gg/NousResearch
- 開源授權：MIT（見 `LICENSE`）

如果你需要英文版詳細說明，請參見根目錄的 `README.md`。
