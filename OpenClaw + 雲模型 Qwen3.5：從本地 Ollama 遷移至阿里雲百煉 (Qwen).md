# OpenClaw + 雲模型 Qwen3.5：從本地 Ollama 遷移至阿里雲百煉 (Qwen)

> **紀錄日期**：2026-03-17
> **適用對象**：受限於 VRAM（如 4GB）導致本地跑 Agent 頻繁崩潰，需處理長文本與複雜任務且不介意使用中國雲服務的同學。

## 前置指南

本指南為地端架構調整。進行雲端 API 遷移前，請確保已完成基礎環境（WSL2、LINE Bot、Cloudflare Tunnel）的架設。若尚未完成的同學，請先參考：

* **[OpenClaw + WSL + Ollama + ngrok + LINE 本地部署完整指南](https://hackmd.io/@80hB8lX2SvWaxYO715T4Ig/S1mGoqz9bx)**
* **[OpenClaw + WSL + Ollama + LINE 本地部署完整指南](https://hackmd.io/@80hB8lX2SvWaxYO715T4Ig/Sk2rXUX5Zg)**

---

## 為什麼選擇阿里雲百煉？

根據前幾天的硬體壓力測試，將後端從本地 Ollama 遷移至阿里雲百煉（Qwen API）主要基於以下三大核心考量：

### 1. 突破硬體 VRAM 瓶頸

* **本地限制**：筆記型電腦 GPU（如 **RTX 3050 Ti 4GB**）在 WSL2 環境下，扣除系統與 Xwayland 佔用的 VRAM 後，可用空間極其有限。
* **Context 困境**：本地執行 `qwen3.5:9b` 時，若設定預設的 262K context，會直接導致 `model failed to load`。即便縮減至 32k，執行 Agent 任務時仍會因空間不足頻繁觸發 `compaction` 失敗。
* **雲端優勢**：百煉 API 支援 **131K (Turbo/Plus)** 甚至 **1M (Flash)** 的超長上下文，完全不佔用本地 VRAM，反應速度比本地運行快 3 至 5 倍。

### 2. 極致的性價比 (2026 定價)

阿里雲百煉目前的定價在 2026 年市場中極具競爭力：

* **Qwen-Turbo**：約為 **¥2 (台幣約 8.8 元) / 1M tokens**。
* **Qwen-Flash**：Input 僅需 **$0.05 / 1M tokens**，是目前市場上最便宜的長文本模型之一。
* **費用計算參考**：

$$Cost = (\frac{Input\_Tokens}{1,000,000}) \times Price_{in} + (\frac{Output\_Tokens}{1,000,000}) \times Price_{out}$$

### 3. 新用戶優惠與台灣友善

* **免費額度**：目前阿里雲國際版提供新用戶高達 **7,000 萬個 Tokens** 的免費試用額度（有效期約 180 天）。
* **國際版支持**：台灣用戶可透過 **阿里雲國際版 (Alibaba Cloud International)** 使用護照實名並綁定一般信用卡，避開中國版繁瑣的認證流程。

---

## 修改指南：將 OpenClaw 接入百煉

### 第一步：取得國際版 API Key

1. 先完成註冊、驗證手機與綁卡：[阿里雲國際版官網](https://www.alibabacloud.com/tc)。
2. 進入 [Model Studio 國際版控制台](https://modelstudio.console.alibabacloud.com/ap-southeast-1#/api-key)。
3. 建立 API Key，Workspace 選擇 **Default Workspace**，取得 `sk-xxx`。

### 第二步：配置 openclaw.json

為了避開百煉對 Tool Schema 驗證過於嚴格導致的 `400 InvalidParameter` 錯誤，必須將 `api` 類型指定為 `openai-completions`。

```bash
# 使用 jq 進行安全寫入，確保不覆蓋原本的 LINE 與 Gateway 安全設定
jq '
  .models.providers.bailian = {
    "baseUrl": "https://dashscope-intl.aliyuncs.com/compatible-mode/v1",
    "apiKey": "你的_API_KEY",
    "api": "openai-completions",
    "models": [
      {
        "id": "qwen3.5-plus",
        "name": "Qwen3.5 Plus",
        "api": "openai-completions",
        "reasoning": false,
        "input": ["text"],
        "cost": { "input": 0.4, "output": 1.2 },
        "contextWindow": 131072,
        "maxTokens": 8192
      }
    ]
  } |
  .agents.defaults.model.primary = "bailian/qwen3.5-plus" |
  .tools.profile = "messaging"
' ~/.openclaw/openclaw.json > /tmp/openclaw_tmp.json && mv /tmp/openclaw_tmp.json ~/.openclaw/openclaw.json

```

### 第三步：修復與啟動

若修改過程中不慎導致 `gateway.mode` 遺失（導致服務崩潰），請執行：

```bash
# 確保 gateway 模式正確
openclaw config set gateway.mode local

# 重啟服務套用設定
openclaw gateway stop && sleep 2 && openclaw gateway start

```

---

## 避坑指南 (Troubleshooting)

* **InvalidParameter 400 錯誤**：這是因為 OpenClaw 傳送了百煉不支援的非標準 JSON Schema（如 `patternProperties`）。
* **對策**：強制使用 `api: "openai-completions"`。
* **工具（Tools）還能用嗎？**：**可以**。在此模式下，OpenClaw 會改用「提示詞模擬」技術。將 `tools.profile` 設為 `messaging` 或 `minimal` 是為了確保模型能精確地透過提示詞邏輯觸發工具，避免複雜 schema 導致的解析失敗。

* **LINE 訊息沒反應**：
* **檢查點 1**：確認 API 類型已正確設定為 `openai-completions`。
* **檢查點 2**：LINE 有 **30 秒強制逾時**，複雜 Agent 任務建議改用 **Web UI** 執行。

* **服務崩潰 (Gateway start blocked)**：手動編輯 JSON 時若遺漏 `gateway` 區塊，服務將無法啟動。
* **修復**：請確保 `openclaw.json` 保留了您原本的強隨機 `token` 與 `channelSecret`。
