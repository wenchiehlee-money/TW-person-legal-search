# TW-person-legal-search

台灣人物法律查核搜尋主控台，以 **Claude Code Skill** 形式運作（不需要 Anthropic API key，
只需要你已登入的 Claude Code 帳號）。

## 怎麼用

打開 Claude Code（`claude` CLI），在對話裡直接打字問就好，不需要輸入指令或 slash command，
Claude 會自動判斷要觸發 `skill-tw-legal-search-console`：

```
幫我查核 https://tw.news.yahoo.com/xxxxx.html   ← 貼一篇提及人物的新聞連結
查一下「王小明 台北」有沒有官司                    ← 姓名 + 地區/公司/職稱等輔助資訊
王小明 是不是 https://example.com/xxx 這篇文章裡的那個人？
王小明 最近有新的裁判紀錄嗎？
```

兩種輸入都可以，Claude 會自動判斷你給的是新聞連結還是姓名，不需要先選模式。**姓名查核強烈
建議附上地區/公司/職稱等輔助資訊**——常見姓名若只給姓名，查到的裁判書常常是完全不相干的
同名人物（甚至是姓名形近/音近但字元不同的人），比對信心只能標「低（同名不確定）」。

若同名候選有多筆裁判書，可以再追問「幫我依線索分組」或指定其中一組要深入查（例如
`我要查 王小明 #2 的詳細清單`），Claude 會依裁判書間的案號互相引用、審級關聯、年齡/住所
等身分細節分組比對，並標註各組之間信心是否足以視為同一人。

第一次使用前，先完成下方「環境需求」的 CLI 安裝（只需做一次）。

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

## 免責聲明

查核結果僅供內部參考，非正式法律或徵信文件；正式用途請洽律師或申請良民證/警察刑事紀錄證明。
系統不會嘗試繞過司法院查詢系統的驗證碼，也不會嘗試回推依法遮蔽的個資。裁判書引用僅限
`skill-tw-legal-rag` 回傳 bundle 的 `allowed_citations`，不得捏造案號。`tw-legal-rag` 後端
可能記錄查詢字串供檢索品質分析，僅送出公開身分資訊，勿送出機密資料。
