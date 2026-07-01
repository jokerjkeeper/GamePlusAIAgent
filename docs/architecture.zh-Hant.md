[English](architecture.md) · **繁體中文** · [简体中文](architecture.zh-Hans.md)

# 系統架構（去識別化）

> 本檔為某商業 Unity 手遊專案 AI Agent 系統的架構快照，已脫敏。
> 整套方案由三大系統協作：**Game Source（遊戲客戶端）**、**Game Agent（知識 Agent 系統，本 repo 展示主體）** 與 **Task（自建工單系統）**。

---

## ① 三大系統與責任邊界

| 系統 | 角色 | 核心責任 | 邊界與解耦 |
|------|------|----------|------------|
| **Game Source**（遊戲客戶端） | 數據來源 | 每日產出 Code Review（Tier1 事實）與規格；接收部署的 agent 設定 | 唯讀部署目標，只供料，不參與策展 |
| **Game Agent**（知識 Agent 系統） | 系統大腦 | 策展成 Wiki 知識庫、需求可行性分析；維護 agent 設定母版並一鍵部署 | 獨立於遊戲客戶端外，可移植至下一專案 |
| **Task**（自建工單系統） | 協作樞紐 | 工單持久化與追蹤；完成單回流 Agent | 與另兩者解耦，僅透過 REST 互動，本身不含 LLM |

```mermaid
flowchart LR
  classDef src fill:#ecfdf5,stroke:#10b981,color:#065f46;
  classDef agent fill:#eef2ff,stroke:#6366f1,color:#3730a3;
  classDef task fill:#fef3c7,stroke:#d97706,color:#92400e;

  PLAN["企劃 / 製作"]
  SRC["Game Source（遊戲客戶端）<br/>每日 Code Review（Tier1）+ 規格"]:::src
  AGT["Game Agent（知識 Agent 系統）<br/>策展引擎 + Wiki 知識庫<br/>需求可行性分析 · agent 設定母版"]:::agent
  TASK["Task（自建工單系統）<br/>工單持久化（REST + SQLite）"]:::task

  PLAN -->|"口語需求"| AGT
  SRC -->|"導入素材（Tier1）"| AGT
  AGT -->|"需求分析 → 開單排程"| TASK
  TASK -->|"Tier2 規格 / 完成單回流"| AGT
  AGT -.->|"設定母版一鍵部署"| SRC
```

---

## ② 資料流管線（產生 → 策展 → 需求分析 → 回流）

```mermaid
flowchart TD
  classDef up fill:#ecfdf5,stroke:#10b981,color:#065f46;
  classDef agent fill:#eef2ff,stroke:#6366f1,color:#3730a3;
  classDef plan stroke-dasharray:5 5,stroke:#d97706,fill:#fef3c7,color:#92400e;

  C["每日 git commit"]:::up --> CR["每日 Code Review（上游 Skill）"]:::up
  CR -->|"(a) 工程回顧報告"| DOCS["工程報告 + 待修清單<br/>（上游自用，不進知識庫）"]:::up
  CR -->|"(b) 知識素材<br/>程式事實 + 檔案:行號，不含評價"| RSRC["上游 codereview 產物"]:::up
  WP["Task 工單系統"] -->|"規格 / 完成單 export"| TIK["raw/tickets"]:::plan
  RSRC -->|"導入"| RAW["raw/codereview"]:::agent
  RAW --> CUR["/curate 清洗引擎"]:::agent
  TIK --> CUR
  CUR --> STATE["(.state)<br/>manifest 增量 · features 仲裁 · review_queue"]:::agent
  CUR --> WIKI["wiki/ 主題頁 + 來源頁"]:::agent
  WIKI -.->|"frontmatter"| BI["build_index.py"]:::agent
  BI --> IDX["INDEX（語意路由表）"]:::agent
  IDX --> ST["/stopic 語意 · 多頁召回"]:::agent
  WIKI --> ASK["/ask 問答"]:::agent
  ST --> REQ["企劃需求 → 動到哪些 .cs<br/>→ 可行性 / plan / 開工單"]:::agent
  ASK --> REQ
  REQ -->|"開立 / 回填"| WP
  CFG["config.json"]:::agent -.->|"paths / sources / tier"| CUR
  CFG -.-> BI
  CFG -.-> ST
```

**兩端職責切分**

| 端 | 角色 |
|----|------|
| 遊戲客戶端 · codereview 產物 | **code 數據**：程式做了什麼 + `檔案:行號`，一天一檔 |
| 遊戲客戶端 · 工程報告 + 待修清單 | **bug / 風險 / 評價**：上游自用，不進知識庫 |
| 知識庫 · `wiki/` | **知識庫**：拆分為「功能 / code」分層，供 AI 按需載入以進行需求分析 |

---

## ③ 目錄結構與職責

```text
agent/                            ← wiki 啟動目錄 = 知識庫本體（Game Agent）
├ CLAUDE.md                       @AGENTS.md（Claude Code 入口）
├ AGENTS.md                       跨工具指令索引 → @.agent/spec/*
├ .agent/
│  ├ config.json                  ★路徑 / 來源 / 權威分級，引擎一律讀取此檔（不硬編碼）
│  ├ commands/                    引擎命令：curate / ask / resolve / stopic
│  └ spec/  git.md  wiki.md       canonical 規範（提交格式 / 拆檔·feature）
├ .claude/
│  └ commands ──junction──▶ .agent/commands   （免提權，讓 Claude Code 識別命令）
├ scripts/
│  └ build_index.py               ★INDEX 自動重生（從各主題頁 frontmatter）
├ raw/        codereview/  tickets/             攝取 inbox
├ .state/     manifest / features / review_queue   增量·仲裁·佇列
└ wiki/       主題-*.md  來源/  INDEX.md        策展知識頁 + 路由表
```

> 此外，Game Agent 另維護一份 **agent 設定母版**（指令 / 命令 / 規範 / 配置），由安裝器一鍵部署至 Game Source；這是「跨專案可移植」的關鍵基礎設施。

---

## ④ 跨工具載入 / 控制鏈（cc · codex · opencode 三家通吃）

```mermaid
flowchart LR
  classDef warn stroke-dasharray:4 4,stroke:#d97706,fill:#fef3c7,color:#92400e;
  CM["CLAUDE.md"] -->|"@AGENTS.md"| AG["AGENTS.md"]
  AG -->|"@import"| SPEC[".agent/spec/<br/>git.md · wiki.md"]
  AG -.->|"不展開 @ → 安裝器內聯"| INL["codex / opencode<br/>指令檔（內聯 spec）"]:::warn
  CC[".claude/commands"] -->|"junction（免提權）"| ACMD[".agent/commands/*.md"]
  CFG["config.json"] -.->|"paths / sources / tier"| ENG["引擎命令<br/>curate · ask · resolve · stopic · build_index"]
```

**三層各用對的機制**

| 層 | 機制 | 為什麼 |
|----|------|--------|
| 指令檔 | `AGENTS.md` 為單一真相，`CLAUDE.md` `@import` | `@import` 免管理員權限；symlink 在 Windows 需提權 |
| 命令 | Windows junction | 目錄連結，建立免管理員權限（symlink 才需要） |
| 相容 | 安裝器內聯 spec | codex / opencode 不展開 `@import` |

---

## ⑤ 權威仲裁狀態機（.state 核心）

```mermaid
flowchart TD
  X["同一 Feature 出現多個來源"] --> D1{"是否存在 Tier1<br/>Code Review?"}
  D1 -->|"是"| OV["高 Tier 覆蓋低 Tier（忽略日期）<br/>被覆蓋者標註『已被實作修正 / 工單取代』"]
  D1 -->|"否 · 同 Tier"| DT["比對來源日期"]
  CF["Code 與工單矛盾<br/>且無 Tier1 可裁決"] --> Q1["review_queue : conflict"]
  UV["工單規格缺乏 Code 佐證<br/>需實作確認"] --> Q2["review_queue : unverified"]
  Q1 --> RS["/resolve 人工處理"]
  Q2 --> RS
```

- **分級**：`Tier1 Code Review（讀取真實程式碼）` ＞ `Tier2 工單規格`。
- **鐵則**：權威分級與定案狀態由人 / 規則決定，**LLM 絕不自行推論**，只負責執行。
- **增量**：依賴 manifest 的 `content_hash`，未變更則略過，不重複策展。

---

## ⑥ INDEX 維護鏈（防漂移）

```mermaid
flowchart LR
  FM["主題-*.md frontmatter<br/>aliases · features · 摘要 · authority"] -->|"build_index.py"| IDX["INDEX（語意路由表）"]
  IDX -->|"LLM 語意判讀"| ST["/stopic 多頁召回"]
  ST --> EX["『技能模擬器』→<br/>編輯器 + TestRunner + 戰場 + GM"]
  TRG["重生時機：頁面變動 / /curate 收尾 / 手動 / --check 防漂移"] -.-> FM
```

INDEX **不准手寫**：由各頁 frontmatter 重生，CI `--check` 過期即 `exit 1`。
