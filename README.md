# TW-person-legal-search

台灣人物法律查核搜尋主控台，以 **Claude Code Skill** 形式運作（不需要 Anthropic API key，
只需要你已登入的 Claude Code 帳號）。支援兩種輸入模式：

1. **依新聞搜尋** — 貼一篇新聞連結
2. **依姓名搜尋** — 輸入姓名（強烈建議附加地區/公司/職稱等輔助識別資訊）

## 快速開始

第一次使用前，先完成下方「環境需求」的 CLI 安裝（只需做一次）。之後在 Claude Code 對話中
直接打字即可，不需要輸入指令或 slash command，例如：

```
幫我查核 https://tw.news.yahoo.com/xxxxx.html
查一下「王小明 台北」有沒有官司
王小明 是不是 https://example.com/xxx 這篇文章裡的那個人？
王小明 最近有新的裁判紀錄嗎？
```

Claude 會自動判斷要觸發 `skill-tw-legal-search-console`，不需要你手動打 `/skill-...`。
若同名候選有多筆裁判書，可以再追問「幫我依線索分組」或指定其中一組要深入查（例如
`我要查 王小明 #2 的詳細清單`），Claude 會依裁判書間的案號互相引用、審級關聯、年齡/住所
等身分細節分組比對，並標註各組之間信心是否足以視為同一人。

## 架構

```
skill-tw-legal-search-console        （主控台，本地整合層，自帶風險分級/輸出格式規則）
  ├─ skill-tw-legal-rag              （裁判書語義檢索，vendored from
  │                                     github.com/aa0101181514/tw-legal-rag）
  │    └─ scripts/search_judgments.py  → 呼叫本機 `twlegalrag` CLI
  │                                       → 打到 https://tlr.dr-legal.com.tw（免 key、
  │                                         2,254 萬+ 筆裁判書語義檢索，附引用防護）
  └─ web search / WebFetch            （新聞佐證、新聞連結擷取當事人脈絡、MOPS 重大訊息，
                                          Claude Code 內建，非本專案技能）
```

`skill-tw-court-records-search`、`skill-tw-person-legal-check`（來自
`github.com/wenchiehlee/skills`）曾是原始設計依賴，但實際查核流程從未真正呼叫它們——
`skill-tw-legal-rag` 的語義檢索完全取代了前者「人工繞司法院驗證碼查詢」的方法，後者的
風險分級規則也已直接內建進 `skill-tw-legal-search-console`。兩者已從本專案與中央 skill
登錄庫移除，避免掛著不會執行的死依賴。

## 目錄結構

- `.claude/skills/` — Claude Code 這個 session 實際載入的技能（與 `skills/` 內容相同）。
- `skills/` — 技能原始碼（可攜版本，方便版本控管與同步到其他專案）：
  - `skill-tw-legal-search-console/` — 本專案的主控台技能，串接下方一個依賴。
  - `skill-tw-legal-rag/` — vendored from
    [aa0101181514/tw-legal-rag](https://github.com/aa0101181514/tw-legal-rag)（License:
    Elastic License 2.0；服務條款見上游 `TERMS.md`）。

## 環境需求

裁判書檢索需要本機安裝 `twlegalrag` CLI（純 Python，免 LLM、免金鑰）：

```bash
python -m pip install --user twlegalrag
```

安裝完成後可用以下指令自我檢查：

```bash
python skills/skill-tw-legal-rag/scripts/search_judgments.py "測試 台北" -n 1
```

若印出 JSON bundle（含 `allowed_citations`、`judgments` 欄位）即代表安裝成功；若出現
`twlegalrag CLI not found`，先確認 `pip install` 有成功完成，再重新開一個終端機/Claude
Code session（見下方 PATH 說明）。

**Windows PATH 注意事項**：若安裝在使用者目錄（`--user`，沒有系統管理員權限時的預設行為），
Windows 上的 `%APPDATA%\Python\Python3XX\Scripts` 目錄通常不在 PATH 裡。本專案已經把該目錄
加進使用者 PATH（**只在下次開新的終端機/Claude Code session 才會生效**，當下這個 session
不會馬上生效）。即使 PATH 沒設好，`skill-tw-legal-rag/scripts/search_judgments.py` 本身也會
自動在該目錄尋找可執行檔，不完全依賴 PATH，通常不需要額外處理。

## 兩種搜尋模式

1. **依新聞搜尋**：先擷取新聞中當事人姓名與脈絡，再進行裁判書查詢與風險分級。
2. **依姓名搜尋**：強烈建議附上輔助識別資訊（地區、公司、職稱等）——常見姓名若未提供，
   查核結果只會標記為「同名不確定」（`skill-tw-legal-rag` 回傳的判決常來自完全不同地區/
   案由的同名人物，甚至姓名形近/音近但字元不同的人，須靠輔助資訊與新聞佐證交叉比對、分組）。

## 免責聲明

查核結果僅供內部參考，非正式法律或徵信文件；正式用途請洽律師或申請良民證/警察刑事紀錄證明。
系統不會嘗試繞過司法院查詢系統的驗證碼，也不會嘗試回推依法遮蔽的個資。裁判書引用僅限
`skill-tw-legal-rag` 回傳 bundle 的 `allowed_citations`，不得捏造案號。`tw-legal-rag` 後端
可能記錄查詢字串供檢索品質分析，僅送出公開身分資訊，勿送出機密資料。
