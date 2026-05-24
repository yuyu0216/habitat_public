# 工程文檔 structure.md

> 《誰動了我的棲地!》前端 + Node 後端 結構索引
> 目的:未來修改時可以一眼定位「哪個畫面 / 元素 / 行為」對應到哪份檔案、哪個 class、哪段 state、哪支 API。
> 變更若影響本表,請順手更新本檔。
>
> **2026-05 架構大轉換**:GAS + Google Sheets 已退場,執行期改為 Node + PostgreSQL + OpenRouter。
> Google Sheets 仍保留為「設計階段」的遊戲資料編輯介面,開局前一次性 seed 到 Postgres,
> 實驗期間 Sheets 不參與任何讀寫。詳見 `~/.claude/plans/ai-delightful-curry.md`。

---

## 一、整體架構速覽

```
[ iPad 瀏覽器 ]              [ 雲端 Node 容器(Render / Zeabur / 自架) ]              [ 外部 ]
   ┌─────────────────┐         ┌─────────────────────────────────────────┐         ┌───────────────┐
   │ src/(純前端)     │ HTTPS  │  Express(server/src/index.js)            │  HTTPS  │ OpenRouter    │
   │ HTML+CSS+原生 JS │ ─────▶ │   ├─ /              靜態前端(src/)        │ ─────▶ │ Claude Haiku  │
   │ state.js 集中狀態 │  REST  │   ├─ /api/game/*    遊戲狀態 + 結算         │  SSE   │ /GPT/Gemini   │
   │ router.js 重畫   │ + SSE  │   ├─ /api/chat      AI 對話(SSE 串流)     │ ─────▶ │ ...           │
   │ #hd-stage        │        │   ├─ /api/teacher/* 老師控制(X-Token)    │         └───────────────┘
   └─────────────────┘        │   ├─ /api/events/*  訊息 polling           │
                              │   └─ /health        健康檢查                │         ┌───────────────┐
                              │                                              │ seed    │ Google Sheets │
                              │  PostgreSQL(同容器)                        │ ◀────── │ (設計期編輯)   │
                              │   執行期 SoR:players/conversations/...      │         └───────────────┘
                              └─────────────────────────────────────────┘
```

- 前端是單頁應用,無打包、無框架,所有腳本以 `<script defer>` 依序載入(見 [src/index.html](src/index.html))。
- 狀態集中在 `window.HD_STATE`(pub/sub),每次 `set()` 觸發 `router.render()` 整塊重畫 `#hd-stage`。
- 高頻欄位(倒數秒數)走 `setSilent()` + 直接 mutate DOM,避免 textarea 因 re-render 掉焦。
- **2026-05-22 轉場集中化**:[main.js](src/js/main.js) 的 `reconcile(rs)` 是所有畫面轉場的唯一入口,依「(round, phase) 是否改變」決定 lobby/action/waiting/report/done;round-status polling 在這四個畫面都跑(間隔 `HD_CONFIG.POLL_ROUND_MS`)。倒數用後端 `round_status.updated_at` 當基準,經 `HD_TIMER.syncTo` 每次 poll 校正,全班對齊。新增 `lobby` 畫面解決「開局前卡回合 0」與「重啟需重整」。
- 舞台固定 1180×820(對齊 PRD 設計基準 / iPad Air 橫向),由 [src/js/main.js:8-16](src/js/main.js#L8-L16) 算 `transform: scale` 適配視窗;viewport meta 用 `width=device-width` 跟著裝置實寬,讓 iPad 不出現水平捲軸。
- **前後端同 origin**:Node 同時供應靜態前端與 API,無 CORS、無 preflight 問題。
- **資料流**:設計階段老師在 Sheets 編 scout_pool / events_pool / traits / species_traits / students,開場前跑一次 `npm run seed` 同步到 Postgres;實驗期間所有讀寫都在 Postgres。
- **LLM**:統一走 OpenRouter,模型由 `LLM_MODEL` 環境變數切換(預設 `anthropic/claude-haiku-4.5`),便於做模型對照實驗。

---

## 二、檔案結構樹

```
game0506/
├── Rule.md                    # 專案規範(技術選擇、命名、工作流)
├── docs/                      # 產品需求文件(實作前必讀 docs/PRD.md)
│   ├── structure.md           # ← 本檔
│   ├── setup.md               # 後端部署 + 操作 + FAQ(不含 Sheet 欄位字典)
│   ├── sheet.md               # Sheet 表格設定字典(v5,10 張 seed 表欄位)
│   ├── mechanic_v5_plan.md    # v5 機制設計與結算公式(當前)
│   ├── v5_data_ready.md       # 可直接填入 Sheets 的完整資料集
│   ├── PRD.md / mechanic_v4.md / ...
│   └── code-review-2026-05-19.md  # bug 評估與修正計畫
├── reference/                 # 設計參考,唯讀,不可修改
├── render.yaml                # Render Blueprint 部署設定(也可移植到其他平台)
│
├── src/                       # 前端實作(由 Node 以靜態檔供應)
│   ├── index.html             # 學生端入口,定義 script 載入順序
│   ├── teacher.html           # 老師端 admin 頁(獨立,不掛 router)
│   ├── changelog.md           # 每次改動往下新增,不刪舊紀錄
│   ├── assets/
│   │   └── map.png            # 四象限棲地底圖(由 CSS background-position 切)
│   ├── styles/                # 同舊版,僅 chat.css 加 initiator 樣式(Stage 8)
│   └── js/
│       ├── config.js          # API_BASE_URL(空字串=同 origin)、MOCK_MODE、timeout
│       ├── session.js         # sessionStorage 包裝(學生資訊持久化)
│       ├── data.js            # 假資料 + 全域常數(物種/棲地/行動/報告)
│       ├── icons.js           # SVG 圖示集 + AP 環產生器
│       ├── state.js           # 全域狀態 + pub/sub(含 chatMessages.initiator、lastConvId)
│       ├── api.js             # REST + SSE 抽象層(login / chat / 各遊戲 API / pollPendingMessages)
│       ├── router.js          # 依 state.screen 切畫面 + 還原 chat 焦點
│       ├── main.js            # 啟動入口 + resize 適配 + session 還原 + 雙 polling loop
│       ├── teacher.js          # 老師端 admin 邏輯(X-Teacher-Token prompt)
│       └── components/        # 同舊版,chat.js 加 initiator label 與 SSE 串流接收
│
├── server/                    # Node 後端(取代整個 gas/)
│   ├── package.json
│   ├── README.md              # 後端本身的快速啟動說明
│   ├── .env.example           # 環境變數範本
│   ├── src/
│   │   ├── index.js           # Express 入口,掛 static + routes + session middleware
│   │   ├── routes/
│   │   │   ├── game.js        # /api/game/*(login/logout/state/habitats/round-status/actions/report/species-traits)
│   │   │   ├── teacher.js     # /api/teacher/*(start-game/progress/settle-round/next-round/reset-game/dev/set-phase)
│   │   │   ├── chat.js        # /api/chat(SSE 串流)
│   │   │   └── events.js      # /api/events/pending(訊息 polling)
│   │   ├── services/
│   │   │   ├── settlement.js  # v5 結算引擎(6 步:三層相乘 + 棲地反饋;見 docs/mechanic_v5_plan.md)
│   │   │   ├── action-formula.js # v5 三層公式共用模組(結算 + 行動預覽共用;computeActionPreview)
│   │   │   ├── scout-filter.js # v4 五層偵查篩選漏斗
│   │   │   ├── filter-parser.js # parseSpeciesFilter / parseHabitatFilter
│   │   │   ├── llm.js         # OpenRouter 呼叫(Claude cache_control + streaming)
│   │   │   ├── prompt-builder.js # persona + 規則 + state + scouts + history
│   │   │   ├── triggers.js    # 預寫腳本選句(round_start/action_explore_auto/scout_result/...)
│   │   │   └── scout-to-dialogue.js # scout 結果包成 system_trigger 訊息(整合在 settlement 內)
│   │   ├── db/
│   │   │   ├── schema.sql     # Postgres 完整 schema(seed 表 + 執行期表 + 對話表)
│   │   │   ├── pool.js        # pg.Pool 實例 + healthCheck
│   │   │   └── queries.js     # 所有 SQL 集中(具名 async 函式)
│   │   ├── middleware/
│   │   │   └── auth.js        # session cookie 驗證,把 student 掛到 req
│   │   └── data/              # 預寫對話腳本(git 版控)
│   │       ├── species-personas.json   # MVP 2 物種人格(green / yellow)+ generic
│   │       ├── dialogue-triggers.json  # 6 種觸發類型
│   │       └── scout-ai-variants.json  # (可選)scout AI 口吻變體
│   └── scripts/
│       ├── seed-from-sheets.js  # npm run seed:Sheets → Postgres(冪等,有 --force)
│       └── export-csv.js        # npm run export:研究結束匯出 6 張表 CSV
│
└── gas/                       # 舊 GAS 後端(已退役,保留供考古)
    └── (參考 git 歷史)
```

---

## 三、啟動與渲染流程

### 載入順序(由 [src/index.html](src/index.html) defer 排序保證)
```
config → session → data → icons → state → api
      → (components) topbar → timer → login → habitat-map
      → side-panel → action-panel → queue-panel → chat
      → action-screen → waiting-screen → survival-report
      → router → main
```

### 啟動(main.js)
1. 讀 `sessionStorage`,若有學生資料直接設 `state.screen = "action"`(刷新不掉)
2. `router.render()` 第一次繪製
3. `HD_STATE.subscribe(router.render)` 訂閱後續變更
4. `window.addEventListener("resize", fitStage)` 適配視窗
5. `HD_TIMER.start()` 啟動全域倒數(每秒 tick)

### Re-render 流程(router.js)
- 觸發點:任何 `HD_STATE.set()` 呼叫
- 機制:`#hd-stage.innerHTML = ""` 後整塊重畫,不做 diff
- 雷區:**chat textarea 會被重建** → router 在清空前 capture `value/selStart/selEnd`,渲染完還原 + `focus()`。修改 router 或 chat 時必須保留此行為(見 [src/js/router.js:8-25](src/js/router.js#L8-L25))。
- 倒數秒數例外:走 [src/js/components/timer.js:21-28](src/js/components/timer.js#L21-L28) 的 `setSilent` + 直接改 pill DOM,不觸發 re-render。

---

## 四、畫面對照表(screen → component)

| `state.screen` | 進入點(router.js)            | 主要元件                                     | CSS                          |
|----------------|---------------------------------|--------------------------------------------|------------------------------|
| `login`        | `hdRenderLogin(state)`           | [login.js](src/js/components/login.js)     | [login.css](src/styles/login.css) |
| `lobby`        | `hdRenderTopBar` + `hdRenderLobby` | [lobby-screen.js](src/js/components/lobby-screen.js)（2026-05-22:等待大廳,內嵌 AI/隊伍 分頁聊天;**2026-05-23 改:lobby 預設 AI、停用隊伍**,後端 `/api/chat` 放寬 guard 讓開局前無 player_state 也能回話) | [lobby-screen.css](src/styles/lobby-screen.css) |
| `action`       | `hdRenderTopBar` + `hdRenderActionScreen` | [action-screen.js](src/js/components/action-screen.js) | [action-screen.css](src/styles/action-screen.css), [habitat-map.css](src/styles/habitat-map.css) |
| `waiting`      | `hdRenderTopBar` + `hdRenderWaiting` | [waiting-screen.js](src/js/components/waiting-screen.js) | [waiting-screen.css](src/styles/waiting-screen.css) |
| `report`       | `hdRenderTopBar` + `hdRenderReport` | [survival-report.js](src/js/components/survival-report.js) | [report-screen.css](src/styles/report-screen.css) |
| `done`         | `hdRenderTopBar` + `hdRenderDone` | [done-screen.js](src/js/components/done-screen.js) | [done-screen.css](src/styles/done-screen.css) |

### 行動階段(action)兩欄組合(平板版 1366×900)
由 [action-screen.js](src/js/components/action-screen.js) 組裝:
```
┌─ TopBar(共用 hdRenderTopBar) ────────────────────────────────────────────┐
├──────────────────────────────────────────────────────┬─────────────────────┤
│ .hd-main                                              │ .hd-side            │
│  ┌──────────────────────────────────────────────────┐ │ 行動選擇 4 張卡       │
│  │ 地圖 .hd-map(2×2,tile 含 badge / name overlay) │ │ (覓食/繁殖/遷移/探索) │
│  └──────────────────────────────────────────────────┘ │  ─────────────────  │
│  ┌──────────┬──────────┬──────────────┐                │ 已選擇佇列 hd-queue  │
│  │ 棲地詳情  │ 我的狀態  │ AI 對話       │                │ ↓                   │
│  │ 資源條    │ 數量/糧食 │ (hd-chat)     │                │ 展開行動 hd-execute │
│  │ + 族群分佈│ /適應+性狀│               │                │                     │
│  └──────────┴──────────┴──────────────┘                │                     │
└──────────────────────────────────────────────────────┴─────────────────────┘
```
- `.hd-shell` 是 `grid-template-columns: 1fr 380px`,gap 14px,padding 14/18/18([action-screen.css](src/styles/action-screen.css))。
- `.hd-main` 是 `grid-template-rows: minmax(0, 1.65fr) minmax(0, 1fr)`。
- `.hd-botrow` 是 `grid-template-columns: 1.2fr 0.9fr 1.5fr`(左卡比較寬,放資源 + 族群分佈 2 欄)。
- `.hd-main` 上下比 `1.4fr / 1fr`,讓底列三張卡有足夠縱向空間。
- AI 對話沿用 `hdRenderChat(state, "action")` 元件(沒有 avatar/bubble 簡化版,直接用我們既有的 hd-chat)。

---

## 五、全域命名空間(window.*)

所有跨檔共用都掛在 `window` 下,**勿改名,勿改全域結構**(會破載入順序的依賴)。

### 設定 / 服務
| 名稱             | 來源檔                                  | 用途                                  |
|------------------|----------------------------------------|---------------------------------------|
| `HD_CONFIG`      | [config.js](src/js/config.js)           | GAS_URL / MOCK_MODE / timeout         |
| `HD_SESSION`     | [session.js](src/js/session.js)         | get/set/clear sessionStorage          |
| `HD_API`         | [api.js](src/js/api.js)                 | `.login()` / `.chat()`                |
| `HD_STATE`       | [state.js](src/js/state.js)             | `.get` / `.set` / `.setSilent` / `.subscribe` |
| `HD_ROUTER`      | [router.js](src/js/router.js)           | `.render()` 整塊重畫                   |
| `HD_TIMER`       | [timer.js](src/js/components/timer.js)  | `.start` / `.stop` / `.reset`         |

### 假資料 / 常數
| 名稱                  | 來源檔                                | 內容                                  |
|----------------------|--------------------------------------|---------------------------------------|
| `HD_SPECIES`         | [data.js](src/js/data.js)             | 6 色族群(id/name/color/hex)         |
| `HD_HABITATS`        | [data.js](src/js/data.js)             | 4 棲地(含 `shortName` 與 `adjacent`)|
| `HD_TRAITS`          | [data.js](src/js/data.js)             | `unlocked[]` / `locked[]`            |
| `HD_PLAYER_DEFAULT`  | [data.js](src/js/data.js)             | 預設玩家(僅備援)                    |
| `HD_GAME`            | [data.js](src/js/data.js)             | 回合 / AP 總量                        |
| `HD_ACTIONS`         | [data.js](src/js/data.js)             | 4 種行動(覓食/繁殖/遷移/探索)       |
| `HD_REPORT`          | [data.js](src/js/data.js)             | 生存報告假資料                        |
| `HD_REPORT_LOGS`     | [data.js](src/js/data.js)             | 偵查報告紀錄假資料(id/round/title/content/unlocked) |

### Render 函式(component → DOM)
| 函式                       | 來源檔                                            |
|---------------------------|---------------------------------------------------|
| `hdRenderLogin`           | [login.js](src/js/components/login.js)           |
| `hdRenderTopBar`          | [topbar.js](src/js/components/topbar.js)         |
| `hdRenderActionScreen`    | [action-screen.js](src/js/components/action-screen.js) |
| `hdRenderHabitatMap`      | [habitat-map.js](src/js/components/habitat-map.js)|
| `hdRenderHabitatDetail`   | [side-panel.js](src/js/components/side-panel.js) — 底列第一張(資源 + 族群分佈 2 欄)|
| `hdRenderMyStatus`        | [side-panel.js](src/js/components/side-panel.js) — 底列第二張(我的狀態 + 已解鎖性狀)|
| `hdRenderActionPanel`     | [action-panel.js](src/js/components/action-panel.js) |
| `hdRenderQueuePanel`      | [queue-panel.js](src/js/components/queue-panel.js)|
| `hdRenderChat(state, mode)` | [chat.js](src/js/components/chat.js)           |
| `hdRenderReport`          | [survival-report.js](src/js/components/survival-report.js) |

### 工具
| 函式                     | 來源檔                              |
|-------------------------|------------------------------------|
| `hdIconSvg(name, size)` | [icons.js](src/js/icons.js)         |
| `hdRoundGaugeSvg`       | [icons.js](src/js/icons.js)         |
| `hdFormatTime(sec)`     | [timer.js](src/js/components/timer.js) |

---

## 六、State Schema

定義在 [src/js/state.js:8-26](src/js/state.js#L8-L26)。所有欄位:

| 欄位               | 型別      | 說明                                                                  |
|-------------------|-----------|----------------------------------------------------------------------|
| `screen`          | string    | `"login"` / `"action"` / `"report"`,driver of router                |
| `player`          | object    | `{ name, id }`,登入後填入                                          |
| `species`         | string    | 玩家扮演的物種 id(畫面不直接揭露名稱)                              |
| `habitatId`       | string    | 當前選中的棲地(`wetland`/`forest`/`urban`/`pond`)                  |
| `round`           | number    | 當前回合                                                              |
| `totalRounds`     | number    | 總回合數                                                              |
| `apTotal`         | number    | AP 總量(每回合上限)                                                |
| `queued`          | array     | 已排入的行動,每筆 `{ id, cost, habitatId, targetHabitatId? }`,可重複 |
| `timerSeconds`    | number    | 倒數剩餘秒數                                                          |
| `timerRunning`    | boolean   | 是否在倒數                                                            |
| `chatMessages`    | array     | `[{ from: "ai"|"user", text, initiator, id? }]`,行動與反思共用;`initiator` ∈ `player`/`auto_send`/`system_trigger`,`id` = conversations.id(polling 去重用) |
| `lastConvId`      | number    | 已收到的最大 conversations.id,訊息 polling 用作 `since` 參數          |
| `chatTyping`      | boolean   | AI 思考中(typing 指示器 + textarea 鎖)                              |
| `reflectionMode`  | boolean   | 進入生存報告即為 true(原本的「先看一般報告再反思」中間步驟已移除,欄位保留作相容) |
| `reportHabitatId` | string?   | 生存報告當前查看的棲地分頁;`null` 代表渲染時自動挑第一個玩家有族群的棲地 |
| `habitatMetrics`  | object    | v5:各棲地當前數值 `{ wetland:{water,vegetation,pollution,max_capacity}, ... }`(0–1),由 `getHabitats().metrics` 寫入,棲地面板用 |
| `actionPreview`   | object    | v5:行動預覽 `{ <habitat>:{ breed:{cost,amount,color}, forage:{cost,food,color}, migrate:{cost,amount,color}, explore:{cost,color} } }`,由 `getActionPreview()` 寫入,行動面板用 |
| `playerFood`      | number    | 我擁有的糧食(mock 寫死 2,之後 GAS 補)                              |
| `playerAdaptation`| string    | 適應程度文字(mock 寫死「良好」,之後 GAS 補)                         |
| `selectedReportLogId` | string? | 生存報告 — 當前選中的偵查報告 id(null = 渲染時自動挑最新一筆)         |
| `reportEventTab`  | string    | 2026-05-22:報告觸發事件分頁 `'A'`(固定回合)/ `'B'`(條件觸發)        |
| `chatTab`         | string    | 2026-05-22:聊天分頁 `'ai'` / `'team'`(lobby + action 用)            |
| `chatDraftAi` / `chatDraftTeam` | string | 2026-05-22:各聊天分頁草稿,切分頁不丟字(輸入即時 setSilent 鏡射)|
| `teamMessages` / `lastTeamMsgId` | array / number | 2026-05-22:隊伍聊天訊息 + polling 游標(`mine` 標自己;樂觀更新靠 id 去重)|
| `reflectionMessages` / `lastReflectionMsgId` / `reflectionDraft` | array / number / string | 2026-05-22:反思機器人聊天訊息 + 游標 + 草稿 |

### 衍生值(不存在 state,渲染時即時算)
- `apRemain = apTotal - sum(queued.cost)` — 由 topbar / action-panel / queue-panel 各自計算([src/js/components/topbar.js:8-10](src/js/components/topbar.js#L8-L10))。

### 變更入口
- `HD_STATE.set({...})` → 通知所有訂閱者(router 重畫)
- `HD_STATE.setSilent({...})` → 只改值不通知(timer.js 專用)

---

## 七、CSS class 對照表(BEM block → 元件)

所有 class 以 `hd-` 為前綴(habitat dashboard),遵循 `Block__Element--Modifier`。

| Block                  | 出現於                                         | 用途                                 |
|------------------------|------------------------------------------------|--------------------------------------|
| `.hd-stage`            | [index.html:21](src/index.html#L21)            | 1180×820 主舞台(由 main.js 套 scale)|
| `.hd-topbar`           | [topbar.js](src/js/components/topbar.js)       | 頂部狀態列                            |
| `.hd-pill`             | base.css / topbar                              | 通用膠囊(姓名/代號/倒數/回合)        |
| `.hd-pill--timer`      | topbar                                         | 倒數膠囊(`--low`/`--zero` 變色)     |
| `.hd-pill--mock`       | topbar                                         | MOCK 模式指示器                       |
| `.hd-roundgauge`       | topbar                                         | 環形 AP 進度                          |
| `.hd-login`            | login.js                                       | 登入卡                                |
| `.hd-shell`            | action-screen.js                               | 行動階段三欄 grid 容器                |
| `.hd-mid`              | action-screen.js                               | 中欄(地圖 + 行動列)                  |
| `.hd-side-l`           | side-panel.js                                  | 左欄棲地細節                          |
| `.hd-detail-card`      | side-panel.js                                  | 卡片骨架(總覽/族群/性狀都用)         |
| `.hd-resource`         | side-panel.js                                  | 食物/水/植被資源條                    |
| `.hd-popmini`          | side-panel.js                                  | 族群分布迷你列                        |
| `.hd-trait`            | side-panel.js                                  | 性狀小膠囊                            |
| `.hd-map`              | habitat-map.js                                 | 2×2 地圖 grid                         |
| `.hd-tile`             | habitat-map.js                                 | 單一棲地 tile(`--{habitat}` modifier)|
| `.hd-tile--active`     | habitat-map.js                                 | 當前選中棲地                          |
| `.hd-species-pin`      | habitat-map.js                                 | 族群圓點(`--self` 含脈動動畫)        |
| `.hd-actbar`           | action-panel.js                                | 行動選擇列容器                        |
| `.hd-action`           | action-panel.js                                | 單張行動卡(`--migrate`/`--disabled`)|
| `.hd-action__sub`      | action-panel.js                                | 遷移卡內的相鄰棲地子按鈕              |
| `.hd-queue`            | queue-panel.js                                 | 右欄已選擇行動容器                    |
| `.hd-queue__item`      | queue-panel.js                                 | 單筆佇列條目(點擊刪除)               |
| `.hd-execute`          | queue-panel.js                                 | 「展開行動」CTA                       |
| `.hd-chat`             | chat.js                                        | AI 對話容器(`data-mode` 區分)        |
| `.hd-chat__msg--ai/user` | chat.js                                      | 訊息氣泡                              |
| `.hd-chat__typing`     | chat.js                                        | AI 思考中三點動畫                     |
| `.hd-report`           | survival-report.js                             | 報告畫面(`--reflection` 切反思模式)  |
| `.hd-report-changes`   | survival-report.js                             | 左半:族群變化清單                     |
| `.hd-change`           | survival-report.js                             | 單筆變化(`--success/--inflow/--negative`)|
| `.hd-event`            | survival-report.js                             | 右半:觸發事件                         |
| `.hd-report-cta`       | survival-report.js                             | 底部「開始反思」/「結束反思」按鈕      |

### CSS 變數(全在 [src/styles/base.css:7-61](src/styles/base.css#L7-L61))
- 棲地色:`--hd-color-{wetland|forest|urban|pond}`
- 族群色:`--hd-species-{red|green|yellow|brown|blue|purple}`
- 紙張/墨色:`--hd-paper`、`--hd-paper-dark`、`--hd-card`、`--hd-card-soft`、`--hd-ink`、`--hd-ink-soft`、`--hd-ink-muted`、`--hd-line`、`--hd-line-soft`
- 強調色:`--hd-accent`、`--hd-warning`、`--hd-danger`、`--hd-success`、`--hd-mist`
- 字體:`--hd-font-sans`(Noto Sans TC)、`--hd-font-display`(Fraunces)、`--hd-font-mono`(JetBrains Mono)
- 圓角:`--hd-radius-{sm|—|lg|xl}` = 7/12/17/24px
- 陰影:`--hd-shadow-{sm|—|lg}`
- 舞台:`--hd-stage-w/-h`(1180/820)、`--hd-topbar-h`(100px)

部分元件透過 inline `style.setProperty("--hd-habitat-color", ...)` 動態套當前棲地主題色(見 [side-panel.js:43](src/js/components/side-panel.js#L43)、[action-panel.js:100](src/js/components/action-panel.js#L100)、[survival-report.js:93](src/js/components/survival-report.js#L93))。

---

## 八、行動 / 佇列邏輯(行動階段核心)

修改行動規則時主要碰這幾支:

| 行為                              | 在哪裡                                                                      |
|-----------------------------------|----------------------------------------------------------------------------|
| 點覓食/繁殖/探索 → 入佇列          | [action-panel.js:36-43](src/js/components/action-panel.js#L36-L43) `pushQueued` |
| 點遷移子按鈕 → 入佇列(含 target)| [action-panel.js:77-87](src/js/components/action-panel.js#L77-L87)        |
| 點佇列條目 → 刪除                  | [queue-panel.js:32-36](src/js/components/queue-panel.js#L32-L36) `removeAt` |
| AP 不足 dim 但仍可刪              | [action-panel.js:22](src/js/components/action-panel.js#L22) `disabled = cost > apLeft` |
| 「展開行動」→ 進入報告             | [queue-panel.js:99-102](src/js/components/queue-panel.js#L99-L102)(Phase B 才接 API)|
| 棲地相鄰關係                       | [data.js](src/js/data.js) 各棲地 `adjacent: [...]` 邊相鄰(不含對角)        |
| 地圖只顯示自己族群 pin             | [habitat-map.js:36-57](src/js/components/habitat-map.js#L36-L57) 用 `playerSpecies` 過濾 |
| 性狀只顯示已解鎖                   | [side-panel.js:89-98](src/js/components/side-panel.js#L89-L98) 不揭露 locked |

---

## 九、API 對照(前端 ↔ Node ↔ Postgres / OpenRouter)

### 前端入口

所有 API 同 origin,前端只看到相對路徑;[`HD_API`](../src/js/api.js) 內部全部用 `fetch + credentials: 'include'` 自動帶 session cookie。

| 呼叫                                | 在哪 trigger                                                            | 對應路由                                  |
|------------------------------------|------------------------------------------------------------------------|------------------------------------------|
| `HD_API.login(name, code)`         | [login.js](../src/js/components/login.js) form submit                  | `POST /api/game/login`                   |
| `HD_API.logout()`                  | 老師端 / 重置                                                          | `POST /api/game/logout`                  |
| `HD_API.getMyState()`              | 行動畫面初始化                                                          | `GET  /api/game/state`                   |
| `HD_API.getHabitats()`             | 行動畫面 + 報告畫面                                                    | `GET  /api/game/habitats`                |
| `HD_API.getRoundStatus()`          | waiting/report polling                                                  | `GET  /api/game/round-status`            |
| `HD_API.submitActions(...)`        | 「展開行動」按鈕                                                        | `POST /api/game/actions`                 |
| `HD_API.getReport(code, round)`    | 進入 report 畫面                                                       | `GET  /api/game/report?round=N`          |
| `HD_API.getSpeciesTraits()`        | 行動畫面顯示物種特性                                                    | `GET  /api/game/species-traits`          |
| `HD_API.getActionPreview()`(P4 接)| 行動面板動態成本/數量 + 變色(v5,後端算)                              | `GET  /api/game/action-preview`          |
| `HD_API.chat(payload)`             | [chat.js](../src/js/components/chat.js) form submit                    | `POST /api/chat`(**SSE 串流**)         |
| `HD_API.pollPendingMessages(since)`| [main.js](../src/js/main.js) 訊息 polling loop                          | `GET  /api/events/pending?since=N`       |
| `HD_API.sendTeamMessage` / `pollTeamMessages`(2026-05-22)| [chat.js](../src/js/components/chat.js) 隊伍分頁 + main.js 訊息 loop | `POST /api/team/chat` / `GET /api/team/messages?since=N` |
| `HD_API.sendReflectionAnswer` / `pollReflectionMessages`(2026-05-22)| [reflection-chat.js](../src/js/components/reflection-chat.js) + main.js 訊息 loop | `POST /api/reflection/answer` / `GET /api/reflection/messages?since=N` |
| 老師端                              | [teacher.js](../src/js/teacher.js)(帶 `X-Teacher-Token` header)        | `POST /api/teacher/{start-game,progress,settle-round,next-round,reset-game}` |

**呼叫規則**(新版):
- `fetch` 用標準 JSON `Content-Type: application/json`(同 origin,無 preflight 問題)
- `credentials: 'include'` 自動帶 session cookie
- `AbortController` + `HD_CONFIG.REQUEST_TIMEOUT_MS`(預設 15s);SSE 不套 timeout
- 失敗統一回 `{ ok: false, error }`,401 由 api.js 觸發登出
- 老師端 403 → 自動清除 X-Teacher-Token 並提示重輸入

### Node 端路由

| 路由                              | handler                                                            | 認證              | 寫入表                          |
|----------------------------------|-------------------------------------------------------------------|-------------------|--------------------------------|
| `POST /api/game/login`           | [routes/game.js](../server/src/routes/game.js)                    | —                 | session                         |
| `POST /api/game/logout`          | 同上                                                              | session           | session                         |
| `GET  /api/game/state`           | 同上                                                              | session           | —                               |
| `GET  /api/game/habitats`        | 同上                                                              | —                 | —                               |
| `GET  /api/game/round-status`    | 同上                                                              | —                 | —                               |
| `POST /api/game/actions`         | 同上(同時寫 `auto_send` 對話訊息)                                | session           | actions / conversations         |
| `GET  /api/game/report`          | 同上                                                              | session           | —                               |
| `POST /api/chat`                 | [routes/chat.js](../server/src/routes/chat.js)(SSE)              | session           | conversations(user + assistant)|
| `GET  /api/events/pending`       | [routes/events.js](../server/src/routes/events.js)                | session           | —                               |
| `POST /api/team/chat` / `GET /api/team/messages` | [routes/team.js](../server/src/routes/team.js)（2026-05-22) | session | team_messages |
| `POST /api/reflection/answer` / `GET /api/reflection/messages` | [routes/reflection.js](../server/src/routes/reflection.js)（2026-05-22) | session | reflection_messages |
| `GET  /api/teacher/player-detail`| 同上(2026-05-23)                                                | X-Teacher-Token   | —(讀:各棲地隻數/糧食/性狀/本回合行動,老師抽屜用) |
| `GET  /api/teacher/settlement-detail` | 同上(2026-05-23)                                            | X-Teacher-Token   | —(讀:某回合全班 reports + trace 計算明細) |
| `POST /api/teacher/start-game`   | [routes/teacher.js](../server/src/routes/teacher.js)              | X-Teacher-Token   | round_status / player_state / conversations(round_start) |
| `POST /api/teacher/settle-round` | 同上                                                              | X-Teacher-Token   | reports / player_state / player_scouts / player_traits / conversations(system_trigger) |
| `POST /api/teacher/next-round`   | 同上                                                              | X-Teacher-Token   | round_status / conversations(round_start) |
| `POST /api/teacher/reset-game`   | 同上                                                              | X-Teacher-Token   | (清空執行期表)                  |
| `GET  /health`                   | [index.js](../server/src/index.js)                                | —                 | —                               |

### Postgres 表結構([server/src/db/schema.sql](../server/src/db/schema.sql))

| 表名                | 來源                | 用途                                                 |
|---------------------|--------------------|-----------------------------------------------------|
| `students`          | Sheets seed         | 學生名單(name / player_code / species / team)     |
| `species_starts`    | Sheets seed         | 物種起始棲地與初始隻數                              |
| `species_traits`    | Sheets seed         | 物種特性(cost / amount / initial_trait;v5 加 food_amount / food_consume / tags) |
| `traits`            | Sheets seed         | 性狀資料庫(v5 擴成 10 欄條件式效果)                |
| `scout_pool`        | Sheets seed         | v4 偵查卡池(含 4 個篩選欄)                         |
| `events_pool`       | Sheets seed         | 事件池(trigger / target_habitat / target_species;v5 加 polution)|
| `habitat`           | Sheets seed(v5)    | 棲地初始 水源/植被/污染/承載量                      |
| `species_action_rate`| Sheets seed(v5)   | 同棲地物種互動 → 行動倍率                            |
| `habitat_change`    | Sheets seed(v5)    | 物種數量 → 棲地數值變化                              |
| `habitat_affect`    | Sheets seed(v5)    | 棲地數值閾值 → 標籤物種增減                          |
| `round_status`      | Node 寫入            | 全域回合狀態(round / phase / updated_at)         |
| `habitat_state`     | Node 寫入(v5)      | 每回合各棲地的 水源/植被/污染(類比 player_state)   |
| `player_state`      | Node 寫入            | 每回合每位玩家的人口/食物/飢餓狀態(JSONB)          |
| `actions`           | Node 寫入            | 學生送出的行動                                      |
| `action_submissions`| Node 寫入            | 防重複送出(UNIQUE(player_code, round))            |
| `player_traits`     | Node 寫入            | 玩家累計解鎖性狀                                    |
| `player_scouts`     | Node 寫入            | 玩家累計取得的偵查情報                              |
| `reports`           | Node 寫入            | 每回合每位玩家的結算報告(2026-05-23 加 `trace` JSONB:結算每步中間值,老師端除錯用;學生 getReport 不取此欄) |
| `conversations`     | Node 寫入            | AI 對話(含 initiator / turn_index / cached_tokens / model_id) |
| `conversation_summaries` | (預留 Stage 10) | 跨回合對話壓縮                                      |
| `team_messages`     | Node 寫入(2026-05-22)| 隊伍聊天(同物種頻道;species/player_code/name/round/message) |
| `reflection_messages` | Node 寫入(2026-05-22)| 反思機器人聊天(role=bot/user;結算推題 + 學生作答) |
| `"session"`         | connect-pg-simple   | 登入 session(cookie 對應)                          |

**seed_status view**:`SELECT * FROM seed_status` 一行確認各表筆數,健康檢查與 seed 後驗證用。

### OpenRouter 呼叫([server/src/services/llm.js](../server/src/services/llm.js))
- baseURL:`https://openrouter.ai/api/v1`(`openai` SDK 相容)
- 模型:由 `LLM_MODEL` 環境變數控,預設 `anthropic/claude-haiku-4.5`
- **Claude prompt caching**:`isClaudeModel()` 偵測到 Claude 時,system prompt 末尾 block 加 `cache_control: { type: 'ephemeral' }`;其他模型自動 caching,marker 被忽略不會出錯
- **Response caching**:`ENABLE_RESPONSE_CACHE=true` 才送 `X-OpenRouter-Cache: true` header;**正式實驗強制 false** 避免污染研究資料,server 啟動偵測 production+true 會印警告
- **Streaming**:`stream: true` + `stream_options: { include_usage: true }` 取 `cached_tokens`
- 失敗 fallback:「我好累…等等再說好嗎?」(寫入 conversations,token 計 0)

### 設計階段:Sheets ↔ Postgres seed

老師仍在 Google Sheets 編輯 6 張表(students / species_starts / species_traits / traits / scout_pool / events_pool),欄位字典詳見 [docs/setup.md](setup.md)。開局前一次性執行:

```bash
cd server
npm run seed        # 冪等,偵測到 live game data 會拒絕
npm run seed -- --force    # 強制覆寫(會清掉執行期資料)
```

seed 腳本流程:讀 Sheets → 驗證必填欄 → 偵測 live data → BEGIN TX → TRUNCATE 設定表 → 逐表 INSERT(每張表自己的 mapper)→ COMMIT → 印筆數摘要。

### 研究資料匯出

```bash
cd server
npm run export                     # 匯到 server/exports/<timestamp>/
npm run export -- /custom/path     # 自訂路徑
```

匯出 6 張 CSV(UTF-8 BOM + RFC 4180 跳脫):`conversations.csv`(研究核心)、`reports.csv`、`player_state.csv`、`player_scouts.csv`、`player_traits.csv`、`students.csv`。詳見 [scripts/export-csv.js](../server/scripts/export-csv.js)。

---

## 十、工程慣例與雷區

### 命名
- class 一律 `hd-` 前綴 + BEM(來自 [CLAUDE.md](CLAUDE.md))
- 全域變數一律 `HD_` 前綴(常數)或 `hdXxx` 前綴(函式)
- 註解用繁體中文

### 重複抽象門檻
- 重複 2-3 次才考慮共用元件,不要一開始就拆(來自 [CLAUDE.md](CLAUDE.md))

### 修改後必做
- 在 [src/changelog.md](src/changelog.md) **往下新增**(不刪舊紀錄),標題格式:`## YYYY-MM-DD 改動主題`

### 雷區 / 易踩坑
1. **Chat textarea 掉焦** — `HD_STATE.set` 會整塊重畫,router 已做 capture/restore;在 chat.js 改 input 或 router.js 改流程時務必保留。
2. **倒數秒數不可走 set** — 必須 `setSilent` + 直改 DOM,否則 textarea 每秒掉焦。
3. **`apRemain` 不在 state 裡** — 由各元件即時算,改 apTotal 或 queued 不要嘗試同步 apRemain。
4. **遷移卡本體不可點** — 點擊邏輯只綁在 `.hd-action__sub` 子按鈕上。
5. **未解鎖性狀不揭露** — 學生只看到後端回傳的 `unlockedTraits`;UI 完全不應出現未解鎖性狀名稱。同樣的不揭露原則也寫進 `prompt-builder.js` 的 HARD_RULES,LLM 不會洩漏。
6. **其他物種 pin 對學生隱藏** — habitat-map 只渲染 `state.species` 自己的 pin。
7. **同 origin、無 CORS** — Node 直接供應 src/ 靜態檔,前端 API 用相對路徑;改部署架構若拆成不同 domain 要重新加 CORS。
8. **conversations turn_index 必須單調** — 防併發:[`getNextTurnIndex`](../server/src/db/queries.js) 用 advisory lock;schema 有 `UNIQUE(player_code, turn_index)` 當最後防線;`listRecentConversations` 用 `ORDER BY id` 不靠 turn_index。
9. **prompt cache 別忘了 marker** — Claude 走 OpenRouter 必須**手動**加 `cache_control: ephemeral`,否則 token 量不會降。`llm.js` 的 `isClaudeModel()` 已包好,改模型清單時要同步維護。
10. **ENABLE_RESPONSE_CACHE 正式必為 false** — 否則 OpenRouter 會回吐快取,污染研究資料。`render.yaml` 已寫死 `false`,自架時務必檢查 `.env`。
11. **舞台縮放是 `transform`** — 子元素用 `position:fixed` 會脫離舞台座標系,需小心;舞台本身用 fixed+translate 居中。
12. **`reference/` 唯讀** — 任何修改都要在 `src/` 或 `server/`,別動參考檔。
13. **seed 是「設計階段」動作** — 實驗開始後不要重跑 seed,會清掉執行期表(player_state / conversations 等)。`scripts/seed-from-sheets.js` 偵測 live data 會拒絕,需 `--force` 才覆寫。

---

## 十一、未來可選優化(MVP 後)

Stage 1-9 已完成,系統可進實驗。以下是後續可選優化:

| 區域                          | 目前狀態                                                | 想做的話                                     |
|------------------------------|--------------------------------------------------------|---------------------------------------------|
| 對話訊息推送                  | 4.5-5.5s polling(已加 jitter,30 人約 6 req/s 無壓力)| 升級 SSE + Postgres LISTEN/NOTIFY 改 <100ms 延遲(無此需求) |
| Postgres → Sheets 反向同步    | 無                                                     | 老師中途想看「目前哪些 scout 被解鎖最多」    |
| Admin UI 取代 Sheets 編輯     | Sheets 是真實編輯介面                                  | MVP 確認後做                                |
| 物種人格擴充                  | MVP 2 物種(green / yellow)+ generic fallback         | 補滿 5-6 物種各自人格                       |
| 對話即時情緒標註              | 無                                                     | 後測分析用,可由 LLM 旁路標記               |
| 監控                          | 無                                                     | Sentry / Grafana / Loki                     |
| conversation_summaries        | schema 已預留,未實作                                  | 回合間壓縮舊對話,讓 prompt 不爆 token       |

未來新增 API 時遵守 [src/js/api.js](../src/js/api.js) 開頭的調用規則(觸發理由、不保險呼叫、不重試、client validate、debounce、合併呼叫、能算就不送)。
