
# VoiceCallbook 分機查詢小幫手（展示版）

> **這是展示版。** 原本是為公司內部同仁做的工具，因為分機表屬於員工個資，這個公開版本裡的姓名、部門與分機全部換成**虛構資料**（80 筆），程式邏輯與原版相同。文件中提到的筆數（例如 1,300 多筆、9 個地區）描述的是原始資料。

公司內部的同仁分機查詢工具。輸入或直接用講的說出同仁姓名，就能查到對方的分機、部門和地區，還可以把查詢結果寄到自己的 email。

查詢比對完全用程式邏輯完成，**不經過任何 AI / LLM 模型**。語音版的語音辨識與語音合成也都在本機執行，不呼叫雲端 API。

## 功能

- **姓名查詢**：輸入「王小明」或「請問王小明分機多少」都可以
- **同名同姓處理**：查到多位同名同姓的人時，會列出每一位的部門與地區，輸入「部門 姓名」就能鎖定是哪一位
- **拼音模糊比對**：語音辨識把名字聽錯成同音字時，用拼音比對還是找得到人，並會先確認「我聽起來像是問 XX」
- **簡繁轉換**：語音辨識輸出的簡體字自動轉成繁體再比對；「臺」和「台」視為同一個字
- **寄送 email**：查完後詢問要不要把結果寄到信箱
- **語音問答**：對著麥克風問，系統用語音回答

## 四種版本

這個 repo 裡有四套可以各自獨立使用的實作，依照需求選擇：

| 版本 | 資料夾 | 適合情境 | 語音 | 寄信方式 |
| --- | --- | --- | --- | --- |
| **Dify 聊天機器人** | 根目錄 `分機查詢小幫手.yml` | 公司已經架好 Dify，只要文字查詢 | 無 | 公司內部 SMTP |
| **Dify + 地端語音** | [`voice/`](voice/) | 需要語音問答，且語音資料不能離開公司 | 本機 SenseVoice + Piper | 透過 Dify |
| **n8n + 地端語音** | [`n8n-voice/`](n8n-voice/) | 想要比 Dify 更輕量的自架平台 | 本機 SenseVoice + Piper | Gmail SMTP 或公司內部 SMTP |
| **純前端網頁** | [`webapp/`](webapp/) | 不想維護任何伺服器，放上靜態網頁空間就能用 | 瀏覽器內建（Web Speech API，會經過 Google） | 開啟 Gmail / Outlook 寄信畫面，由使用者自己按送出 |

每個資料夾都有自己的 README，寫了詳細的安裝步驟、限制和取捨。

## 技術架構

| 元件 | 技術 |
| --- | --- |
| 對話流程 | [Dify](https://dify.ai/) Chatflow（Python Code 節點）或 [n8n](https://n8n.io/) workflow |
| 語音辨識（STT） | [SenseVoice-small](https://github.com/FunAudioLLM/SenseVoice)（FunASR），常駐的本機 Python 服務，在 CPU 上約 0.1～0.2 秒 |
| 語音合成（TTS） | [Piper](https://github.com/rhasspy/piper)（`zh_CN-huayan-medium` 中文女聲），常駐行程，約 1.5 秒 |
| 簡繁轉換 | [opencc-js](https://github.com/nk2028/opencc-js) |
| 拼音比對 | [pinyin](https://github.com/hotoo/pinyin) |
| 語音伺服器 | Node.js |

語音辨識原本使用 Whisper-small，實測 SenseVoice 對真實姓名的準確度相同，速度快了約 10～15 倍，因此改用 SenseVoice。

## 專案結構

```
.
├── 分機查詢小幫手.yml   # Dify Chatflow，可直接匯入（內嵌分機資料）
├── 使用說明.md          # Dify 版的匯入步驟、SMTP 設定與網路注意事項
├── voice/               # 地端語音引擎（STT、TTS、查詢邏輯）+ Dify 版語音介面
├── n8n-voice/           # n8n 版：workflow 定義、匯入腳本、語音介面
├── webapp/              # 純前端版：index.html + data.js
└── scripts/             # 把各地區分機表 Excel 整理成資料庫的程式
```

## 快速開始

**Dify 版**：在 Dify 選「透過 DSL 檔案建立」，匯入 `分機查詢小幫手.yml`，再設定 SMTP 環境變數。完整步驟請看 [使用說明.md](使用說明.md)。

> Dify 預設的 sandbox 容器網路是隔離的，可能連不到公司郵件伺服器，上線前請先和 IT 確認網路設定，細節寫在使用說明裡。

**語音版**：

```bash
cd voice
npm install
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt
# 另外需要下載 Piper 執行檔與中文語音模型，請見 voice/README.md
node server.js
```

第一次啟動時會自動下載 SenseVoice 模型（約 900 MB）。

## 更新分機資料

總機提供新的分機表時，用 `scripts/` 裡的程式重新整理，就能一次更新所有版本使用的資料。步驟請看 [scripts/README.md](scripts/README.md)。

## 資料與隱私

分機資料包含同仁姓名、部門與分機，屬於公司內部資訊：

- Dify 版與語音版的查詢、寄信都在公司內部完成，資料不會送到外部雲端
- 純前端版會把整份分機資料放在網頁裡，任何能打開網頁的人都看得到，只能部署在公司內部或有登入保護的網頁空間
