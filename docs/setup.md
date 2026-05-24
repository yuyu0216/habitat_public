# 後端部署與操作手冊

> 老師端的「部署 + 操作 + FAQ」一份搞定。
> 目標讀者:第一次接手這個系統的老師 / 工程師,跟著走就能讓遊戲跑起來。
>
> **2026-05 架構轉換**:後端已從 GAS + Google Sheets 改為 Node + PostgreSQL + OpenRouter。
> Google Sheets 退到「設計階段的編輯介面」——老師仍在 Sheets 編平衡、編觸發、編人格,
> 開局前跑一次 `npm run seed` 把資料同步到 Postgres,實驗期間 Sheets 不參與任何讀寫。
>
> **結算機制為 v5**(棲地驅動 × 性狀生效 × 物種互動,見 [docs/mechanic_v5_plan.md](mechanic_v5_plan.md))。
> **每張 Sheet 表的欄位怎麼填,請看 [docs/sheet.md](sheet.md)**——那是表格設定的唯一字典,本檔不再重複。

---

## 目錄

1. [Node 後端部署與腳本總覽](#一node-後端部署與腳本總覽)
2. [設計階段資料:Sheets → Postgres seed](#二設計階段資料sheets--postgres-seed)
3. [執行期 Postgres 表(系統寫入,老師別動)](#三執行期-postgres-表系統寫入老師別動)
4. [常見問題](#四常見問題)

---

## 一、Node 後端部署與腳本總覽

### 1.1 部署選項

支援以下平台,擇一即可。詳細 server 端啟動說明見 [server/README.md](../server/README.md)。

| 平台              | 適用                              | 設定檔                           |
|-------------------|----------------------------------|----------------------------------|
| Render            | 雲端託管(免費 dev、$7/月 starter) | [render.yaml](../render.yaml)    |
| Zeabur            | 雲端託管(新專案需付費,~$5/月)    | 用 Node + PostgreSQL service     |
| 自架(Ubuntu + Tailscale 等) | 教室筆電 / NUC                | 自己跑 `npm install` + `npm start` |

部署架構都一樣:**Node web service + PostgreSQL**,Node 同時供應前端靜態檔與所有 API。

### 1.2 第一次啟動流程(任一平台共通)

```
1. clone repo,進 server/ 目錄,跑 npm install
2. 起 PostgreSQL,記下 DATABASE_URL
3. cp .env.example .env,填:
   - DATABASE_URL          → Postgres 連線字串
   - SESSION_SECRET        → 隨機 32+ 字元(node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")
   - TEACHER_PASSWORD      → 老師端登入用(亦充當 X-Teacher-Token)
   - OPENROUTER_API_KEY    → 從 openrouter.ai dashboard 取得
   - LLM_MODEL             → 預設 anthropic/claude-haiku-4.5
   - ENABLE_RESPONSE_CACHE → 開發 true 可,正式實驗務必 false
   - GOOGLE_SA_JSON        → Google Service Account JSON 檔的絕對路徑(seed 用)
   - SHEET_ID              → Google Sheets 的 ID
4. 建 schema:psql $DATABASE_URL -f src/db/schema.sql
5. 在 Google Sheets 填學生名單與遊戲資料(欄位字典見 docs/sheet.md)
6. 跑 npm run seed → 看到「seed 完成!」+ 各表筆數
7. npm start(正式)或 npm run dev(檔案改動自動重啟)
8. 開瀏覽器訪問 http://<server-host>:3000/health 確認 ok
9. 老師端訪問 http://<server-host>:3000/teacher.html,輸入 TEACHER_PASSWORD
```

### 1.3 改了 Node 程式碼之後

- **自架**:重啟 server(若用 `npm run dev` 會自動重啟)
- **Render / Zeabur**:`git push origin main` → 平台自動部署
- **新增了 npm 套件時**:務必重跑 `npm install`(2026-05-24 起依賴 `archiver` 做匯出 ZIP;雲端平台部署會自動 install,自架要手動跑)
- **schema.sql 改了**:對開發 DB 跑 `psql -f src/db/schema.sql`(`IF NOT EXISTS` 不會覆蓋舊表;檔尾遷移段用 `ADD COLUMN IF NOT EXISTS`,重跑安全);若要清空重來請手動 DROP

#### 1.3a 新增 Sheet 欄位 / 更新既有 DB(2026-05-24 起)
本系統**沒有自動 migration**。新增欄位(如 2026-05-24 的 `impact_desc` / `metric*` / `question` / `bonus_ap` / `explore_count`)要走完整四步:
1. **clean install(實驗常態)**:整個 DB 重建時,直接 `psql -f src/db/schema.sql` 會一次帶出所有欄(CREATE 段已含新欄),再 `npm run seed -- --force` 即可。
2. **更新既有 DB(不想重建)**:**重跑** `psql $DATABASE_URL -f src/db/schema.sql` 即可——檔尾的 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` 會把缺的欄補進現有表(不動既有資料)。Render/Zeabur 可在 DB console 貼這段 ALTER,或本機對 `DATABASE_URL` 跑。
3. **Sheet 端**:在 Google Sheet 工具列跑「🛠 棲地設定 → 補欄(安全)」把新欄補進對應分頁(不洗資料),填內容。
4. **同步**:`npm run seed -- --force`(或老師頁「🔧 刷新參數」二次確認)把 Sheet 新欄灌進 Postgres。
> 順序重點:**先 ALTER(步驟 2)再 seed**——表上沒有該欄時 seed 的明確欄位 INSERT 會直接報錯。

### 1.4 可用的 npm scripts

| 指令 | 用途 | 何時跑 |
|---|---|---|
| `npm start` | 正式啟動 server | 部署後 |
| `npm run dev` | 開發模式(Node 20+ 內建 --watch) | 開發時 |
| `npm run seed` | Sheets → Postgres 同步(冪等,偵測 live data 會拒絕)。**亦可改用老師頁「🔧 刷新參數」按鈕** | 開局前 / 改了 Sheets 後 |
| `npm run seed -- --force` | 強制覆寫(會清掉執行期 player_state / conversations / habitat_state 等)。**老師頁刷新參數遇 live data 二次確認 = 此效果** | 重置實驗時 |
| `npm run export` | 匯出 6 張 CSV 到 `server/exports/<timestamp>/`。**亦可改用老師頁「⬇ 匯出研究資料」按鈕(回 ZIP 下載)** | 課後做研究 |
| `npm run export -- /custom/path` | 自訂匯出目錄 | 同上 |

> CLI 與老師頁按鈕共用同一份邏輯([server/src/services/seed.js](../server/src/services/seed.js) `runSeed`、[server/src/services/export-data.js](../server/src/services/export-data.js) `buildAllCsvs`),兩條路結果一致。

### 1.5 老師端控制方式

老師端在 [src/teacher.html](../src/teacher.html),首次操作會 prompt 輸入 `TEACHER_PASSWORD`,之後 admin API 自動帶 `X-Teacher-Token` header。

| 動作 | admin 頁(teacher.html) | 直接打 API(curl / Postman) |
|---|---|---|
| 刷新參數(Sheets→Postgres seed) | 「🔧 刷新參數(Sheet→DB)」按鈕(2026-05-24) | `POST /api/teacher/seed`(body `{force}`) |
| 開場初始化 | 「⚠ 開場 startGame」按鈕 | `POST /api/teacher/start-game` |
| 重啟一場 | 「🔄 重啟本場」按鈕 | `POST /api/teacher/reset-game` |
| 看進度 | 進度頁面格子 | `GET /api/teacher/progress` |
| 結算 | 「結算本回合」按鈕 | `POST /api/teacher/settle-round` |
| 下一回合 | 「下一回合」按鈕 | `POST /api/teacher/next-round` |
| 匯出研究資料(6 張 CSV 打包 ZIP) | 「⬇ 匯出研究資料(CSV)」按鈕(2026-05-24,常駐隨時可匯) | `GET /api/teacher/export` |
| dev 強切 phase | (沒按鈕) | `POST /api/teacher/dev/set-phase`(dev 環境用) |

**所有老師端 API 都要帶 `X-Teacher-Token: <TEACHER_PASSWORD>` header**。

> **2026-05-24 新增**:seed 與 export 現在老師頁有按鈕,不必進 terminal——
> - 「🔧 刷新參數」= 網頁版 `npm run seed`(偵測到 live data 會跳二次確認,確認後等同 `--force`)。
> - 「⬇ 匯出研究資料」= 網頁版 `npm run export`,但直接回傳 ZIP 下載(不寫伺服器磁碟)。
> - 老師頁還新增**回合倒數**顯示(action 階段;與學生端同基準 `ROUND_DURATION_SEC`)。
> - 結算明細抽屜新增 **habitat_change**(物種數量→棲地數值)摘要,可驗證該條有跑、數字怎麼來。

---

## 二、設計階段資料:Sheets → Postgres seed

老師在 Google Sheets 編「設計階段」的遊戲資料,開局前跑 `npm run seed` 同步到 Postgres。實驗期間所有讀寫都在 Postgres,Sheets 不參與。

**👉 每張表的每一欄要填什麼,完整字典在 [docs/sheet.md](sheet.md)。** 本節只列總覽。

### 2.1 seed 表總覽(老師可編輯)

v5 共 **10 張** seed 表,對應 Postgres 同名表:

| 分頁名 | 群組 | 用途 | 必填? |
|---|---|---|---|
| `students` | 誰在玩 | 學生名單(登入用) | ✅ 必填 |
| `species_starts` | 誰在玩 | 物種起始棲地與初始隻數 | ✅ 必填 |
| `species_traits` | 物種基礎值 | 行動 AP/數量、覓食倍率、食物消耗、標籤(v5 移除 forage_power,加 food_amount/food_consume/tags) | ✅ 必填 |
| `traits` | 修正 | 性狀(v5 從純顯示變「會生效」,擴成 10 欄條件式) | ✅ 重點 |
| `species_action_rate` | 互動 | 同棲地物種互動 → 行動倍率(v5 新) | ✅ 重點 |
| `habitat` | 棲地 | 棲地初始 水源/植被/污染/承載量(v5 新,取代寫死值) | ✅ 必填 |
| `habitat_change` | 棲地 | 物種數量 → 改棲地數值(v5 新) | ⬜ 可選 |
| `habitat_affect` | 棲地 | 棲地數值過閾值 → 標籤物種增減(v5 新) | ⬜ 可選 |
| `events_pool` | 事件 | 人類事件(v5 加 polution 欄、物種棲地閾值 trigger) | ✅ 重點 |
| `scout_pool` | 事件 | 探索卡牌(v4 起含 4 個篩選欄) | ✅ 可調 |

> **退役分頁**:`species_prompts`(物種人格改放 [server/src/data/species-personas.json](../server/src/data/species-personas.json))、`game_state`(已棄用)、舊 GAS 時代的 `conversations` Sheet(已遷到 Postgres)。seed 不會讀這些。

### 2.2 seed 流程

```bash
cd server
npm run seed              # 冪等,偵測到 live game data 會拒絕
npm run seed -- --force   # 強制覆寫(會清掉執行期資料)
```

seed 腳本流程([server/scripts/seed-from-sheets.js](../server/scripts/seed-from-sheets.js)):
讀 Sheets(6 張既有表用嚴格讀、4 張 v5 新表用容錯讀,缺分頁當空表)→ 翻譯層把中文翻成 id → 驗證必填欄 → 偵測 live data → BEGIN TX → TRUNCATE 設定表 → 逐表 INSERT → COMMIT → 印筆數摘要。

驗證:`SELECT * FROM seed_status;` 或開 `/health`,一行確認各表筆數。

---

## 三、執行期 Postgres 表(系統寫入,老師別動)

這些表由 Node 在遊戲進行中寫入,老師**不需要也不應該手動改**,但理解欄位有助於課後看資料。完整 schema 見 [server/src/db/schema.sql](../server/src/db/schema.sql),結構索引見 [docs/structure.md §九](structure.md)。

| 表 | 寫入時機 | 老師看得到的線索 |
|---|---|---|
| `round_status` | 回合/phase 切換 | 當前回合與階段(`idle`/`action`/`waiting`/`settling`/`report`/`done`) |
| `habitat_state` | startGame + 每回合結算 | 每回合各棲地的 水源/植被/污染(v5,類比 player_state) |
| `player_state` | 每回合結算後 | 每位玩家各棲地族群分布(JSONB)、食物、飢餓狀態 → 看誰從哪回合開始衰退 |
| `actions` | 學生送出行動時 | 誰最積極、哪棲地最熱門、哪行動最受歡迎 |
| `action_submissions` | 學生送出行動時 | 防重複送出(UNIQUE per player+round) |
| `player_traits` | 解鎖性狀時 | 玩家累計解鎖的性狀(同性狀只記第一次) |
| `player_scouts` | 抽到探索卡時 | 玩家累計取得的偵查情報(防重複) |
| `reports` | 每回合結算後 | 每位玩家結算報告;`changes`/`events`/`scouts`/`new_traits`/`trace` 為 JSON |
| `conversations` | 每次 AI 對話 | 研究核心:含 initiator / turn_index / cached_tokens / model_id / response_ms |
| `conversation_summaries` | (預留) | 跨回合對話壓縮 |
| `team_messages` | 隊伍聊天 | 同物種頻道訊息 |
| `reflection_messages` | 報告反思 | 機器人推題 + 學生作答 |
| `"session"` | 登入 | connect-pg-simple 自動管理的登入 session |

**phase 切換流程**:
```
idle(初始)→ startGame → action(學生排行動)
  → 老師按結算 → settling(處理中,通常 < 2 秒)
  → report(學生看報告)→ 老師按下一回合 → action(round +1)
  → ... 重複 → done(第 5 回合報告看完)
```

**老師可手動操作的特例**:想踢某學生下線(讓他在新裝置登入)→ 刪 `"session"` 表該玩家的 row,或 `POST /api/teacher/reset-game` 清空執行期狀態。

---

## 四、常見問題

### Q1: 改了 Sheets 之後,Postgres 不會自動更新嗎?

A: 不會。Sheets 是「設計階段」的編輯介面,**必須跑 seed** 才會同步到 Postgres。
做法二擇一(2026-05-24 起):**老師頁「🔧 刷新參數」按鈕**(免進 terminal)或 CLI `npm run seed`。
- 若實驗還沒開始(player_state / conversations 都還沒資料),直接按「刷新參數」/ 跑 `npm run seed`
- 若實驗已經有資料,seed 會拒絕(避免覆寫),這時要評估:
  - 不想動實驗資料 → 不要改 Sheets(或改了也不 seed,改動不生效)
  - 真的要重來 → 老師頁刷新參數的**二次確認**(等同 `--force`)或 CLI `npm run seed -- --force`(會清掉 player_state / conversations / habitat_state 等執行期表)

### Q2: 改了 events_pool 結算沒套用

A: events_pool 在 Node 結算時是**從 Postgres 讀**,不是即時從 Sheets 讀。所以:
1. 先在 Sheets 編輯 events_pool
2. 跑 `npm run seed`(或 `--force`,看狀況)
3. 結算才會用到新事件

注意事項:
- target_species 留空 = `<any>`,不會出錯
- trigger 拼錯字(例 `round_eq=1` 應為 `round_eq:1`)會靜默失敗,沒任何提示——seed 時可注意 log
- `polution` 欄拼字就是少一個 l,Sheets 標頭要照打,否則污染永遠 0

### Q3: 改了 species_traits 前端沒反映

A: species_traits 是學生**登入時拉一次**,改完要請學生重新整理頁面才會生效。
**最保險的做法**:Sheets 改完 → `npm run seed -- --force`(實驗前)→ 老師端按「🔄 重啟本場」→ 學生重登入。

### Q4: 重複登入訊息一直出現,但我只開了一個 tab

A: Postgres 的 `"session"` 表(connect-pg-simple 自動管理)可能有殘留 session。可手動刪該玩家的 row,或執行 `POST /api/teacher/reset-game` 清空執行期狀態。

### Q5: 學生發現 AP 點數跟其他人不一樣

A: 這是設計,每物種 AP cost 不同(`species_traits`)。v5 還會因「性狀 × 當前棲地數值」動態調整成本。如果想讓所有物種一樣,把 species_traits 的 4 個 cost 欄改成相同數字,再跑 `npm run seed`。

### Q6: 想新增第 7 種物種

A: 物種清單 hardcode 在前端([src/js/data.js](../src/js/data.js) 的 `HD_SPECIES`)與後端([server/src/services/filter-parser.js](../server/src/services/filter-parser.js) 的 `parseSpeciesFilter`)。新增需要改前後端,不能只改 Sheet。

### Q7: 我想看每回合每位玩家的詳細變化

A: 老師頁按「⬇ 匯出研究資料(CSV)」(隨時可匯)或 CLI 跑 `npm run export`,匯出的 `reports.csv` 與 `player_state.csv` 就是完整紀錄。
- `reports.csv` 的 `changes` / `events` / `scouts` / `new_traits` 都是 JSON 字串
- 用 Excel / Google Sheets 開 CSV(有 BOM 不會亂碼),也可用 Python pandas 解析 JSON 欄
- 想看單一回合「逐棲地 / 逐步計算」的中間值:老師頁進度格點學生 → 結算明細抽屜(含 habitat_change)

### Q8: 結算很慢(超過 5 秒)

A: 正常情況 < 2 秒(Postgres 比 Sheets 快得多)。若超過 5 秒:
- 檢查 server log 有無錯誤
- 確認 Postgres 連線健康(`/health` endpoint)
- 30 人同時送 action 不會卡(Node async + Postgres 無 lock 問題)
- 老師端按按鈕後不要連點,等 alert 出來

### Q9: OpenRouter API 額度爆掉怎麼辦?

A: 短期應急:
- 切換到便宜的模型:`LLM_MODEL=openai/gpt-4o-mini` 或 `google/gemini-flash-1.5`,重啟 server
- 暫停實驗,等下個月 quota reset

長期:在 OpenRouter dashboard 設 budget cap,提早收到通知。

### Q10: 為什麼正式實驗不能開 response cache?

A: `ENABLE_RESPONSE_CACHE=true` 會讓 OpenRouter 把「相同輸入回相同輸出」直接從快取吐回,不再呼叫 LLM。對開發省成本很好,但對研究(分析 AI 親和力如何影響學生)是**致命污染**:同一段話會得到一模一樣的回應,完全沒有個別差異。`render.yaml` 已寫死 `false`,自架時務必確認 `.env` 也是 `false`,server 啟動會印警告若被誤開。

---

## 附錄:檔案對應表

- 本文件:`docs/setup.md`(部署 + 操作 + FAQ)
- **Sheet 欄位字典:[docs/sheet.md](sheet.md)**(表格設定的唯一字典)
- 機制設計:[docs/mechanic_v5_plan.md](mechanic_v5_plan.md)(v5,當前) / [mechanic_v4.md](mechanic_v4.md)(v4) / [mechanic_v3.md](mechanic_v3.md) / [mechanic.md](mechanic.md)(v2)
- 可直接填入的完整資料集:[docs/v5_data_ready.md](v5_data_ready.md)
- 試跑與調參:[docs/B8_tuning_guide.md](B8_tuning_guide.md)
- 程式結構:[docs/structure.md](structure.md)
- 改動紀錄:[src/changelog.md](../src/changelog.md)
- 後端原始碼:[server/src/](../server/src/)(Express routes + services + db)
- 後端腳本:[server/scripts/](../server/scripts/)(seed / export)
- 前端原始碼:[src/index.html](../src/index.html) / [src/teacher.html](../src/teacher.html) / [src/js/](../src/js/) / [src/styles/](../src/styles/)
- GAS 程式:`gas/sheet_setup.gs` **仍在用**(Google Sheet 選單「⚠ 全部重建」= `rebuildAllSheets`,重建設定表為 DATA_V3 預設、students 名單保留);`gas/Code.gs` / `gas/phase_b.gs` 為退役 runtime(保留供考古,不再部署)。三個檔皆保留,清空+重建 Sheet 走 `rebuildAllSheets`,不需刪 .gs
</content>
</invoke>
</invoke>
