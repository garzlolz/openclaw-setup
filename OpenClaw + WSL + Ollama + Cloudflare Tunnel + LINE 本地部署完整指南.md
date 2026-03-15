# OpenClaw + WSL + Ollama + Cloudflare Tunnel + LINE 本地部署完整指南

> 紀錄日期：2026-03-15
>
> 環境：Windows 11 + WSL2 (Ubuntu) + i7-12700h (筆電) + NVIDIA RTX 3050 Ti 4GB + RAM 32GB

---

## 環境架構

```
Windows 11
└── WSL2 (ubuntu_openclaw)
    ├── Ollama（背景服務，port 11434）
    │   ├── qwen3.5:4b
    │   ├── qwen3.5:9b
    │   ├── qwen3.5-4b-16k（自訂 16k context 版本）
    │   └── qwen3.5-9b-16k（自訂 16k context 版本）
    ├── OpenClaw（gateway，port 18789）
    │   └── Web UI：http://localhost:18789
    └── cloudflared（Webhook 轉發，port 18789 → HTTPS 固定網址）
        └── line.<你的域名>.dpdns.org
```

---

## 安裝步驟

### 1. 建立專用 WSL Distro

```powershell
# 若沒有現有 distro，先安裝 Ubuntu
wsl --install Ubuntu

# 備份乾淨環境
wsl --export <乾淨的 distro 名稱> <匯出位置>

# 匯入為 OpenClaw 專用 distro
wsl --import <distro 名稱> <安裝位置> <匯入位置>

# 啟用
wsl -d <你的 distro 名稱>
```

### 2. 更新系統與安裝套件

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential curl git wget zip unzip npm zstd jq
```

### 3. 透過 nvm 安裝 Node.js v22

> OpenClaw 需要 Node.js 22+，系統預設版本通常不夠新。

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 重新載入 shell 後執行
nvm install 22
nvm use 22
nvm alias default 22
```

**讓每次開新終端機自動切到 v22：**

```bash
# 確認 nvm 初始化已在 .bashrc（應在第 119~121 行附近）
grep -n "nvm" ~/.bashrc

# 在 nvm bash_completion 那行後面插入自動切換
sed -i '/bash_completion/a nvm use 22 --silent' ~/.bashrc
source ~/.bashrc
```

### 4. 安裝 Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh

# 確認能正常執行
ollama
```

> 執行 `ollama serve` 時若出現 `address already in use`，代表服務已在背景運作，屬正常現象。

### 5. 拉取模型

```bash
# 推薦先拉 4b（較快驗證環境）
ollama pull qwen3.5:4b

# 確認可正常運作
ollama run qwen3.5:4b "hello"

# 之後再拉 9b
ollama pull qwen3.5:9b
```

### 6. 建立自訂 16k context 模型

> 必要步驟，OpenClaw 預設 contextWindow 262144 會導致 VRAM 不足載入失敗。

**4b 版本：**

```bash
cat > ~/Modelfile4b << 'EOF'
FROM qwen3.5:4b
PARAMETER num_ctx 16384
EOF

ollama create qwen3.5-4b-16k -f ~/Modelfile4b
```

**9b 版本：**

```bash
cat > ~/Modelfile9b << 'EOF'
FROM qwen3.5:9b
PARAMETER num_ctx 16384
EOF

ollama create qwen3.5-9b-16k -f ~/Modelfile9b
```

### 7. 安裝並啟用 OpenClaw

```bash
ollama launch openclaw
```

互動介面裡選擇模型（建議先選 `qwen3.5:4b`），OpenClaw 會自動設定 `openclaw.json` 並備份。

啟動成功後終端機會顯示：

```
✓ OpenClaw is running
  Open the Web UI:
    http://localhost:18789/#token=ollama
```

> 若出現 npm 相關錯誤，確認 nvm 的 Node.js v22 已正確設為 default。

> **重要：** `ollama launch openclaw` 安裝完後，`openclaw` CLI 的路徑不會自動加進 PATH。需完成步驟 3 的「自動切換 v22」設定，否則每次開新終端機都會出現 `openclaw: command not found`。

### 8. 修正 OpenClaw 設定

**8-1. 將 contextWindow 改為 16384：**

```bash
openclaw config set models.providers.ollama.models.contextWindow 16384
```

**8-2. 更新模型 id/name 為自訂版本：**

```bash
sed -i 's/"id": "qwen3.5:4b"/"id": "qwen3.5-4b-16k"/' ~/.openclaw/openclaw.json
sed -i 's/"name": "qwen3.5:4b"/"name": "qwen3.5-4b-16k"/' ~/.openclaw/openclaw.json
openclaw config set agents.defaults.model.primary ollama/qwen3.5-4b-16k
```

**8-3. 關閉 thinking mode（避免回應過慢）：**

```bash
sed -i 's/"reasoning": true/"reasoning": false/' ~/.openclaw/openclaw.json
```

**8-4. 重啟 gateway 套用設定：**

```bash
openclaw gateway stop && sleep 2 && openclaw gateway start
```

### 9. 串接 LINE Official Account

#### 9-1. 在 LINE Developers Console 建立 Messaging API Channel

1. 前往 [LINE Developers Console](https://developers.line.biz/console/)
2. 選擇或建立 Provider
3. 點「**Create a Messaging API channel**」
4. 填入必要資訊後送出
5. 建立完成後，LINE 會自動在 [LINE Official Account Manager](https://manager.line.biz/) 產生對應的官方帳號
6. 回到 Developers Console，進入剛建立的 channel
7. 「**Basic settings**」分頁 → 複製並記錄 **Channel Secret**
8. 「**Messaging API**」分頁 → **Channel access token** → 點「**Issue**」產生並複製

#### 9-2. 安裝 LINE Plugin

```bash
openclaw plugins install @openclaw/line
```

#### 9-3. 設定 config

```bash
openclaw config set channels.line.channelAccessToken "你的_channel_access_token"
openclaw config set channels.line.channelSecret "你的_channel_secret"
```

確認寫入：

```bash
cat ~/.openclaw/openclaw.json | jq '.channels'
```

#### 9-4. 設定 Cloudflare Tunnel（取代 ngrok）

LINE Webhook 必須是 HTTPS，使用 Cloudflare Tunnel + 免費域名提供固定不變的 HTTPS 網址。

**前置：取得免費域名**

1. 前往 [https://dash.domain.digitalplat.org](https://dash.domain.digitalplat.org) 以 GitHub 帳號註冊並完成 KYC
2. 申請 `.dpdns.org` 免費域名（每年需在到期前 180 天內免費續期）
3. 前往 [https://dash.cloudflare.com](https://dash.cloudflare.com) → **Domains** → **Add a site**，輸入域名，選 **Free 方案**
4. 取得 Cloudflare 分配的兩個 Nameserver，回 DigitalPlat 填入 NS1/NS2

**安裝 cloudflared（直接下載 binary，避免 GPG key 問題）：**

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared
sudo mv cloudflared /usr/local/bin/
cloudflared --version
```

**建立 Tunnel：**

```bash
# 登入 Cloudflare（會開啟瀏覽器授權）
cloudflared tunnel login

# 建立命名式 tunnel
cloudflared tunnel create openclaw-line

# 設定 DNS 子域名指向 tunnel
cloudflared tunnel route dns openclaw-line line.<你的域名>.dpdns.org
```

**建立設定檔：**

```bash
# 將 <TUNNEL_ID> 換成 tunnel create 時顯示的 UUID
sudo mkdir -p /etc/cloudflared
cat > /etc/cloudflared/config.yml << 'EOF'
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: line.<你的域名>.dpdns.org
    service: http://localhost:18789
  - service: http_status:404
EOF

# 複製憑證到 /etc/cloudflared/
cp ~/.cloudflared/<TUNNEL_ID>.json /etc/cloudflared/
```

**設為背景服務（開機自啟）：**

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

`active (running)` 即代表成功。

#### 9-5. 填入 Webhook URL

1. 前往 Developers Console → 你的 channel →「**Messaging API**」分頁
2. 找到「**Webhook URL**」欄位，填入：

```
https://line.<你的域名>.dpdns.org/line/webhook
```

> **注意：** 路徑必須是 `/line/webhook`，不是 `/webhooks/line`。

3. 點「**Verify**」確認連線正常（DNS 需生效後才能成功，通常幾分鐘內）
4. 開啟「**Use webhook**」開關

#### 9-6. 設定 LINE Official Account Manager

前往 [https://manager.line.biz/](https://manager.line.biz/) → 你的帳號 → **Settings → Response settings**：

- **Chat** → **關閉**
- **Webhooks** → **開啟**
- **Auto-response messages** → **關閉**

> 這步驟容易漏做，未開啟 Webhooks 的話 LINE 訊息不會送到 OpenClaw，gateway log 也不會有任何反應。

#### 9-7. 重啟 Gateway 並完成配對

```bash
openclaw gateway stop && sleep 2 && openclaw gateway start
```

從 LINE 傳訊息給 bot，會收到類似：

```
OpenClaw: access not configured.
Your lineUserId: Uxxxxxxxxx
Pairing code: XXXXXXXX
Ask the bot owner to approve with:
openclaw pairing approve line XXXXXXXX
```

執行配對：

```bash
openclaw pairing approve line <配對碼>
```

完成後即可透過 LINE 與 OpenClaw AI 對話。

---

## 遇到的問題與解法

### 問題一：`LLM request timed out` / `model failed to load`

**症狀：**
OpenClaw Web UI 顯示 `Ollama API error 500: model failed to load`。

**原因：**
OpenClaw 預設 `contextWindow: 262144`（262K tokens），要求 Ollama 分配大量 KV cache，遠超過 4GB VRAM 可負擔的量。Xwayland（WSL GUI 服務）額外佔用 600MB~1.5GB，讓可用 VRAM 更少。

**查看 Ollama log 確認：**

```bash
journalctl -u ollama -f
```

Log 會看到 Ollama 不斷嘗試從 7 個 GPU layers 降到 1，最後全部 offload 失敗。

**解法：** 執行步驟 6 建立 16k 自訂模型，並依步驟 8 更新設定。

---

### 問題二：thinking mode 造成回應過慢

**症狀：**
TUI 底部顯示 `think low`，每次回應前都有大量推理過程。

**原因：**
`openclaw.json` 裡模型的 `reasoning` 預設為 `true`。

**解法：**

```bash
sed -i 's/"reasoning": true/"reasoning": false/' ~/.openclaw/openclaw.json
openclaw gateway stop && sleep 2 && openclaw gateway start
```

> 注意：`openclaw config set agents.defaults.model.thinking false` 對此版本會報 validation error，須改用 `sed` 直接修改 json。

---

### 問題三：`100% context used` 對話卡住

**症狀：**
Web UI 底部出現紅色警告 `100% context used`，模型無法繼續回應。

**解法：**
開新對話，點左側「聊天」旁的 **+** 按鈕，或直接在瀏覽器輸入：

```
http://localhost:18789/chat?session=new
```

---

### 問題四：切換模型後 Web UI 仍顯示舊模型名稱

**說明：**
這是 UI 快取問題，實際使用的模型以回應訊息下方的 tag 為準。開新對話後 tag 就會正確顯示新模型名稱。

---

### 問題五：`ollama serve` 報 `address already in use`

**說明：** 這不是錯誤，代表 Ollama 服務已在背景運作，可直接進行後續步驟。

---

### 問題六：`openclaw: command not found`

**原因：**
`openclaw` CLI 安裝在 nvm 管理的 node 路徑下，開新終端機時若 nvm 沒有自動切到 v22，PATH 就不包含 openclaw。

**解法：**

```bash
grep "nvm use 22" ~/.bashrc
```

若沒有，手動加入：

```bash
sed -i '/bash_completion/a nvm use 22 --silent' ~/.bashrc
source ~/.bashrc
```

---

### 問題七：LINE Webhook Verify 回傳 404

**原因：** Webhook URL 路徑填錯。

**正確路徑：**

```
https://line.<你的域名>.dpdns.org/line/webhook
```

**錯誤路徑（不可用）：**

```
https://line.<你的域名>.dpdns.org/webhooks/line
```

---

### 問題八：LINE 訊息沒有進到 OpenClaw

**症狀：**
Verify 成功（200 OK），但從 LINE 傳訊息後 cloudflared log 沒有收到任何 POST，gateway log 也沒反應。

**原因：**
LINE Official Account Manager 的 **Webhooks 開關未開啟**。

**解法：**
前往 [https://manager.line.biz/](https://manager.line.biz/) → Settings → Response settings → 把 **Webhooks** 開關打開。

---

### 問題九：`sudo cloudflared service install` 找不到設定檔

**症狀：**
```
Cannot determine default configuration path. No file [config.yml config.yaml] in [~/.cloudflared ...]
```

**原因：**
`sudo` 以 root 身份執行，找不到一般使用者家目錄下的設定檔。

**解法：**
將設定檔複製到 `/etc/cloudflared/` 後再安裝：

```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/
sudo cp ~/.cloudflared/<TUNNEL_ID>.json /etc/cloudflared/
sudo sed -i 's|/home/<你的使用者名稱>/.cloudflared/|/etc/cloudflared/|g' /etc/cloudflared/config.yml
sudo cloudflared service install
```

---

### 問題十：cloudflared log 出現 `context canceled` / `control stream failure`

**說明：**
cloudflared 預設維持 4 條連線，單一連線暫時斷線會自動重連，屬正常現象。只要 LINE Bot 能正常回應訊息，這些 WRN/ERR 可忽略。

---

## 硬體限制與模型選擇

| 模型                          | 大小   | 4GB VRAM            | 備註               |
| :---------------------------- | :----- | :------------------ | :----------------- |
| `qwen3.5:4b`（預設 262k ctx） | 3.4 GB | 無法載入            | contextWindow 太大 |
| `qwen3.5-4b-16k`（16k ctx）   | 3.4 GB | 可完整放入          | **推薦，速度快**   |
| `qwen3.5:9b`（預設 262k ctx） | 6.6 GB | 無法載入            | contextWindow 太大 |
| `qwen3.5-9b-16k`（16k ctx）   | 6.6 GB | 部分 offload 到 RAM | 速度較慢但效果較好 |

> RTX 3050 Ti 只有 4GB VRAM，Xwayland 會額外佔用 600MB~1.5GB，務必確認剩餘 VRAM 足夠再載入模型。

---

## 切換模型

```bash
# 切換到 9b
sed -i 's/"id": "qwen3.5-4b-16k"/"id": "qwen3.5-9b-16k"/' ~/.openclaw/openclaw.json
sed -i 's/"name": "qwen3.5-4b-16k"/"name": "qwen3.5-9b-16k"/' ~/.openclaw/openclaw.json
openclaw config set agents.defaults.model.primary ollama/qwen3.5-9b-16k

# 切換回 4b
sed -i 's/"id": "qwen3.5-9b-16k"/"id": "qwen3.5-4b-16k"/' ~/.openclaw/openclaw.json
sed -i 's/"name": "qwen3.5-9b-16k"/"name": "qwen3.5-4b-16k"/' ~/.openclaw/openclaw.json
openclaw config set agents.defaults.model.primary ollama/qwen3.5-4b-16k

# 重啟套用
openclaw gateway stop && sleep 2 && openclaw gateway start
```

> 切換後須開新對話，回應 tag 才會正確顯示新模型名稱。

---

## 常用指令速查

```bash
# 查看目前 VRAM 使用量
nvidia-smi

# 確認 Ollama 模型列表
ollama list

# 強制釋放模型佔用的記憶體
ollama stop qwen3.5-4b-16k

# 重啟 OpenClaw gateway
openclaw gateway stop && sleep 2 && openclaw gateway start

# 查看 OpenClaw gateway log
journalctl --user -u openclaw-gateway.service -f

# 查看 Ollama log
journalctl -u ollama -f

# 開啟 TUI
openclaw tui

# 讓 gateway 在關閉終端機後繼續運作
sudo loginctl enable-linger $USER

# 查看 Cloudflare Tunnel 狀態
sudo systemctl status cloudflared

# 重啟 Cloudflare Tunnel
sudo systemctl restart cloudflared

# 查看 Cloudflare Tunnel log
sudo journalctl -u cloudflared -f
```

---

## 備註

- OpenClaw 設定檔位置：`~/.openclaw/openclaw.json`
- OpenClaw Web UI：`http://localhost:18789/#token=ollama`
- Node.js 版本需要 22+，建議透過 nvm 管理，避免系統套件版本過舊
- 新對話入口：左側「聊天」旁的 **+** 按鈕，或網址 `http://localhost:18789/chat?session=new`
- LINE Webhook URL 格式：`https://line.<你的域名>.dpdns.org/line/webhook`
- Cloudflare Tunnel 設定檔位置：`/etc/cloudflared/config.yml`
- dpdns.org 域名每年需在到期前 180 天內免費續期，到期不續則域名失效
- LINE Official Account Manager 的 Webhooks 開關需手動開啟，與 Developers Console 的 Use webhook 是兩個不同設定
- cloudflared log 出現 `context canceled` 屬正常現象，不影響功能