# Session 接力檔 — GamePlusAIAgent 公開作品集

> 用途：在 `d:\Git\GamePlusAIAgent` 目錄開 Claude Code 時，讀此檔即可接續進度。
> 最後更新：2026-06-29。

---

## 一句話目標

把某商業 Unity 手遊專案的 AI agent 系統，做成一份**對外公開、脫敏的「設計巧思」作品集 README**（履歷展示用），繁體中文。

## 來源素材（在另一個 repo，**只讀、不要改**）

來源專案：`D:\OlgCase\RS\HSLR_Agent`，主要素材：

- `docs/架構.md` + `docs/架構.html` — 手寫設計快照，五節 + mermaid（**主素材**）。
- `docs/architecture.html` — `/architecture` 自動生成，系統全貌 / 模組表 / 技術棧。
- `AgentPlan.md` — 設計取捨的「為什麼」（§1.1 工具實際行為、§1.2 claudeMdExcludes、§2.3 砍掉 design skills 實測僅 30%）。
- `README.md` — 既有定位句。

## 已拍板的決策（來自與使用者的討論）

| 議題 | 決定 |
|------|------|
| 受眾 / 目的 | **對外作品集 / 履歷展示**（非操作手冊，重點在設計巧思與取捨） |
| 公開性 | **公開，需脫敏**（商業遊戲） |
| 放置位置 | 全新空目錄 `d:\Git\GamePlusAIAgent`（未來推上 GitHub） |
| 語言 | **繁體中文** |
| README 結構 | **折衷**：保留常規 README 外殼與順序，但把「安裝/運行手冊」換成「系統如何運作（概念）」+「專案狀態」 |
| 定位 | **純展示**，不裝成可安裝框架 |
| demo | 未來若有人詢問，**另開一個 GitHub repo** 做可實際運行的 demo（脫敏起手包，類似「使用者把 github 套到自己專案」的流程）。**現在不做**。 |
| Karpathy 參考 | 借「狠 hook 措辭 + 第一人稱 + 極簡不堆 badge」；**不**借他「可跑產品/install/run」的目錄骨架 |

## 脫敏政策（每一處都要套）

| 原始 | 公開版 |
|------|--------|
| `HSLR` / `OlgCase` / 真實遊戲名 | 「某商業 Unity 手遊專案」/「遊戲客戶端」（**全匿名**） |
| 絕對路徑 `D:/OlgCase/RS/...` | 相對 / 泛化 |
| `api.conf` token、TaskHub `:5003`、UnityMCP `:8088` | 移除；僅以「自託管工單系統 / MCP」概念帶過 |
| 真實 commit hash、`.cs` 行號 | 化名（`Foo.cs:120-140`）或標「示意」 |
| 精確數量（46 raw / 100 feature / 26 頁） | 用「數十天 / 數十頁」模糊化 |

> 脫敏自檢：grep 不應命中 `HSLR|OlgCase|D:/|:5003|:8088|TaskHub|wepages|game_brain`（`.gitignore` 內的 `.claude/api.conf` 是刻意忽略，非洩漏）。

## 目前成品（本目錄）

**已三語化（EN 預設 / 繁中 / 簡中），共 9 個 .md：**

```
GamePlusAIAgent/
├ README.md            ← 英文（GitHub 預設）；系統層敘事 9 節，含三系統拓撲圖
├ README.zh-TW.md      ← 繁體中文
├ README.zh-CN.md      ← 簡體中文
├ docs/
│  ├ architecture.md / architecture.zh-TW.md / architecture.zh-CN.md   ← 系統架構六節（①三系統責任邊界 ②資料流含回流 ③目錄 ④跨工具 ⑤仲裁 ⑥INDEX），多張 mermaid
│  └ design-notes.md / design-notes.zh-TW.md / design-notes.zh-CN.md   ← 引擎層深掘（五巧思 + 一個「不做」決策）；**待收斂補 anti-RAG / LLM 不推論**
├ .gitignore
└ .claude/session.md   ← 本檔
```

**多語慣例（career-ops 風格）**：
- 每檔第 1 行為語言切換列：`[English](…) · [繁體中文](…) · [简体中文](…)`，當前語言**粗體無連結**，其餘為連結。
- 切換列後空一行接 H1。
- 內文交叉連結指向**同語言** docs（EN→`docs/*.md`、繁→`*.zh-TW.md`、簡→`*.zh-CN.md`）。
- README H1 後加一行**個人化斜體 tagline**（career-ops 取經，配合第一人稱方向）。
- 視覺維持極簡：**只加語言切換列 + tagline，不加 badge/hero**（延續 Karpathy 決策）。
- 英/簡兩語由 subagent 平行產出，繁中由主程式建；mermaid 僅譯引號內標籤，節點 ID／classDef／箭頭原樣。

**⚠️ 大改版（2026-06-29 第二輪）：敘事從「引擎層」重構為「系統層」**
原 README 把重心放在策展引擎內部（三層格式/仲裁/INDEX/去重），使用者回饋「與架構出入大、見樹不見林」。
經源頭核對（Explore agent 比對 `D:\OlgCase\RS\HSLR_Agent` 的 `arch.md`/`AgentPlan.md`/`gametar/README.md` 等），確認真正要講的是**三大系統協作**，已據此重寫 README + architecture（design-notes 引擎文件這輪未動）。

**三大系統（對外化名 ← 真實內部名，真實名禁出現）**：
| 對外化名 | 真實（禁出現） | 角色 / 責任 |
|----------|---------------|------------|
| Game Source（遊戲客戶端） | HSLR_3 | 數據源：每日 codereview(Tier1)+規格；接收部署的 agent 設定 |
| Game Agent（知識 Agent 系統） | HSLR_Agent | 大腦：策展成 wiki、需求分析；維護「設定母版」一鍵部署；**可移植** |
| Task（自建工單系統） | TaskHub | 工單持久化(REST+SQLite)；與兩者解耦、不含 LLM；完成單回流 |

**新 README 章節**：① 標題+tagline → ② 核心痛點(使用者三條:AI 優化製程/AI 把需求進工單/迭代 wiki 讓下個專案快啟) → ③ 三大系統與責任邊界(表+拓撲 mermaid) → ④ 系統如何運作(Data Lifecycle 7 步,新增第 7 步「回流」) → ⑤ 設計巧思**系統層** A/B/C + 🔧 引導去 design-notes → ⑥ 架構(引導去 architecture) → ⑦ 技術棧 → ⑧ 專案狀態 → ⑨ 脫敏聲明。

**設計巧思分兩層**：
- **系統層**（README ⑤，新頭條）：A. 自建工單(捨 Jira/Trello/Mantis,刻意極簡解耦、AI 全留 Agent 側;**保守貼源頭**,不說「AI 從零開發」) · B. Agent 解耦+設定母版可移植 · C. 三系統數據飛輪(規格 Tier2 + 完成單回流)。
- **引擎層**（搬去 design-notes 深掘，README 只用 pointer 帶過）：單檔三層 / 權威仲裁 / 跨工具 / INDEX 防漂移 / 增量去重。

**資料流關鍵更新**：Task ⇄ Agent **雙向回流**——工單規格(Tier2)進策展;**完成單 export 回 Agent 再 /curate 提煉**(較新功能,舊源頭未記,使用者口述為準)。

**脫敏清單擴充**（第二輪冒出的真實 token，全域 grep 須零命中）：
`HSLR|OlgCase|TaskHub|gametar|wepages|WEPAGES|game_brain|127.0.0.1|:5003|:5002|:8088|D:/`

## 狀態

- ✅ README + architecture（三語 6 檔）已**重構為系統層敘事**、已脫敏（擴充清單全域 grep 乾淨）。
- ✅ 9 檔語言切換列一致、互連正確；mermaid 結構不動只譯標籤。
- ✅ 繁中語氣已改為**正式台式工程書面語**（見記憶 zh-tw-engineering-register；以 Gemini 版為基準清乾淨）。
- ✅ **design-notes（引擎文件）已收斂**（三語）：標題改為「知識引擎設計筆記」；新增 §1「為何不用 RAG」（把結構化提前到 write-time，含三病根表：結構被碾平 / 定案分不出 / 來源決策靠猜）+ §3 子節「鐵則：LLM 絕不自行推論」；三語 README 引導句已同步補上這兩個主題。前身 RAG 系統一律泛稱（不出現真名），脫敏 grep（含 `ragNovel`）零命中。
- ✅ **已發布 GitHub**：remote `https://github.com/jokerjkeeper/GamePlusAIAgent.git`，分支 `main`（2026-06-29）。
  - commit 作者已設為 **jokerjkeeper `<4235965+jokerjkeeper@users.noreply.github.com>`**（ID-based noreply，計入貢獻圖、連個人頁、不外洩 gmail）；已寫進 repo local config。
  - 三語 README「規劃中/Planned」已加 **可運行 Demo Repo**（去識別化起手包，另開 repo）Roadmap 條目。
  - GitHub repo **About 描述 + topics 已填**（topics: ai-agent / claude-code / game-development / knowledge-base / knowledge-curation / llm / mermaid / unity / documentation / portfolio）。
  - 最新 commit `49b3326`（roadmap）；commit 歷史乾淨（無 Claude trailer）。
  - `.gitignore` 已擴充為忽略**整個 `.claude/`**（session.md 含未脫敏內部資訊，絕不可公開）。
  - push 前已用 `git grep --cached` 掃描追蹤檔：禁用詞**零命中**；確認 `session.md`/`api.conf` 未被追蹤。
  - 追蹤檔 = 10 個（`.gitignore` + 3 README + 6 docs）。
- ✅ **Demo repo 已建立並推送**：`https://github.com/jokerjkeeper/GamePlusAIAgent-starter`（目前設 **private**），分支 `main`，commit `7974e80`，作者 jokerjkeeper(ID-noreply)。本機目錄 `D:\Git\GamePlusAIAgent-starter`。
  - 結構：三系統三目錄 **`game-agent/`**（HSLR_Agent 機制：引擎+結構+脫敏 `build_index.py`/`install.py`）、**`game-source/`**（只 cr-daily skill）、**`task/`**（只 verify：`verify.md` + 最小 `acceptance.sql`）。共 30 檔。
  - **方針**：搬「機制」(可移植腳本/規範,脫敏)不搬「真實語料」——raw/wiki/.state 全用**合成範例**(generic 功能,無 幻世錄/Steam ID/角色/真 .cs)。
  - 可實跑:`game-agent` 下 `python scripts/build_index.py`(+`--check`)、`install.py --dry-run`、`cr_collect.py`、`acceptance.sql`。LLM 命令以 spec 形式收錄。
  - **README 已三語化**（commit `d30251e`）：頂層 + 三子目錄各 EN/繁/簡，共 12 個 README，含語言切換列；繁中用台式工程書面語。操作/程式檔（SKILL.md/verify.md/commands/scripts）依範圍維持英文。
  - 推送需**互動認證**(GCM)：Bash 工具非互動會卡 `could not read Username`，由使用者自行 `git push` 完成（每次 push 都要使用者跑）。
  - **脫敏新增高危項(來自三來源探索,公開時務必擋)**:真實遊戲名 `幻世錄`、Steam AppID `3803940/4030150/4778780`、角色 `雷歐納德/緹娜公主`、技能 `極光之劍`、`HSLR_Agent/.claude/api.conf` 內 **live token `wep_…`**、`wepages`、`D:/Dev/Program/wepages`、`WEPAGES_*` env、Discord webhook、真 `.cs`(SteamConfig/TeleportSystem/NewUIDialogueTalk…)。
- ⏳ **主 repo 公開 Roadmap 的 starter 實連結「暫不加」**：starter 為 private,連過去會 404；待使用者日後轉公開再補連結。
- ⏳ （可選）starter 之後可補：三語 README、轉公開、`install.py` 完整化、rule-based `/curate` stub。

## 待辦 / 下一步（接手時可挑）

1. **語氣微調**：✅ 已做一輪「加重第一人稱」（2026-06-29）。策略：集中在 `design-notes.md`（部落格體的家），README 只在開頭鉤子輕點一筆。已注入 5 處第一人稱「決策當下卡點」：design-notes 引言、#1 兩份文檔、#2 仲裁日期踩雷、#3 symlink 權限坑、附錄「不做」Excel。脫敏 grep 仍乾淨。若仍嫌輕/重，可再調；目前刻意維持「卡點才用第一人稱、結構說明維持客觀」的分工。
2. **推上 GitHub**：`git init` → 建 repo → 首次 commit / push（使用者親自做；可請 AI 給指令）。
3. **demo repo**（未來、另起）：脫敏可運行起手包，常規 README（install/run/usage 住這裡），README 末尾補一行 `▶ 想自己跑跑看？見 demo repo` 串接。
4. （可選）README 開頭是否加 badge / hero 圖 / 目錄 / 英文 tagline——目前皆無，維持極簡。

## 注意事項

- 編輯時務必持續套**脫敏政策**，新增內容也要 grep 自檢。
- 來源 `HSLR_Agent` 只作素材**唯讀**，不要改它。
