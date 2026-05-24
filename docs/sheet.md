# Google Sheets 表格設定字典(v5)

> **這份文件只講一件事:老師要在 Google Sheets 把每張表的「每一欄」填成什麼。**
> 部署、啟動、結算引擎、FAQ 請看 [docs/setup.md](setup.md);機制設計脈絡看 [docs/mechanic_v5_plan.md](mechanic_v5_plan.md)。
>
> 對應的程式真相來源:[server/src/db/schema.sql](../server/src/db/schema.sql)(欄位定義)、
> [server/scripts/seed-from-sheets.js](../server/scripts/seed-from-sheets.js)(讀法與容錯)。
> 本文件若與這兩支程式衝突,以程式為準,並請順手修正本檔。

---

## 0. 30 秒看懂

- 老師在 **Google Sheets** 編 10 張「設計階段」表 → 開局前跑一次 `npm run seed` → 資料進 **Postgres** → 實驗期間 Sheets 不再參與讀寫。
- v5 是「**三層相乘**」模型:**最終值 =(物種基礎值 + 性狀條件修正)× 物種互動倍率**,且棲地數值(水源/植被/污染)每回合會被物種與事件改變。
- 因此你會編到的東西分四群:
  1. **誰在玩**:`students` / `species_starts`
  2. **物種基礎值**:`species_traits`
  3. **修正與互動**:`traits`(性狀條件)/ `species_action_rate`(物種互動)
  4. **棲地與事件**:`habitat`(棲地初值)/ `habitat_change`(物種改棲地)/ `habitat_affect`(棲地害物種)/ `events_pool`(人類事件)/ `scout_pool`(探索卡)

```
①基礎       species_traits        繁殖基數 / 覓食倍率 / food_consume ...
   ├②條件修正  traits              看「當前棲地數值」決定 +/-
   └③互動倍率  species_action_rate 同棲地特定物種共存 → ×百分比(可相乘)
   最終值 =(① + ②)× ③
④棲地反饋   habitat_change(物種→棲地)、habitat_affect(棲地→物種)、events_pool(人類事件)
```

---

## 1. 共通規則(填任何表前先讀)

### 1.1 id 對照(老師可填中文,seed 會翻成 id)

seed mapper 內建翻譯層([seed-from-sheets.js:128-158](../server/scripts/seed-from-sheets.js#L128-L158)),**填中文或 id 都可以**,引擎一律用 id。

| 類別 | 中文 → id |
|---|---|
| 物種 | 紅火蟻→`red`、綠鬣蜥→`green`、貢德氏赤蛙/赤蛙→`yellow`、美國牛蛙/牛蛙→`brown`、領角鴞→`blue`、蓋斑鬥魚/鬥魚→`purple` |
| 棲地 | 南部農田/濕地農田/農田→`wetland`、淺山森林/森林→`forest`、都會公園/公園→`urban`、埤塘水域/埤塘→`pond` |
| 棲地數值 metric | 水源→`water`、植被→`vegetation`、污染/汙染→`pollution` |
| 行動 action | 覓食→`forage`、繁殖→`breed`、遷移→`migrate`、探索→`explore` |
| 變量 variable | 食物→`food`、行動點/點數→`AP`、數量→`AM` |

> 建議:Sheets 端用「資料驗證 → 下拉(來自範圍)」鎖中文選項,打錯字直接被擋。

### 1.2 物種篩選語法(`target_species` / `species` 等欄通用)

| 寫法 | 意義 |
|---|---|
| `<any>` 或留空 | 所有物種 |
| `red` | 只紅火蟻 |
| `red,green,blue` | 多物種,半形逗號分隔 |
| `<any-except:purple>` | 除了紫色都影響 |
| `<any-except:red,blue>` | 多物種例外 |
| `aqua` | 用**標籤**選整群物種(見下) |
| `aqua,red` | 標籤 + 物種可混用 |
| `<any-except:bird>` | 排除某標籤整群 |

#### 用標籤(tag)當「整群物種」的捷徑

`target_species` 可直接填 **`all` / `aqua` / `land` / `bird`**,引擎會展開成帶該標籤的所有物種(標籤定義見 §1.6):
- `aqua` → yellow, brown, purple(水生)
- `land` → red, green, yellow, brown(陸生)
- `bird` → blue
- `all` → 全部(等同 `<any>`)

> ⚠️ **標籤展開目前只在 `events_pool.target_species` 生效**(人類事件)。`traits.species` 只認單一物種 id 或 `<any>`;`species_action_rate` 用 `cond_species_*` 逐一指定物種,都不吃標籤。

### 1.3 棲地篩選語法(`target_habitat`)

`wetland` / `forest` / `urban` / `pond` 之一,或 `<any>`(全部棲地)。

### 1.4 閾值規則(`cond_threshold` / `threshold`)

**正數 = 嚴格大於、負數 = 嚴格小於、等於時不套用**。
例:`0.3` 代表「> 0.3」、`-0.3` 代表「< 0.3」。

### 1.5 數值制式

| 用在哪 | 制式 | 範例 |
|---|---|---|
| 棲地數值 water/vegetation/pollution、`habitat_change.value` | **0–1 小數制** | `0.6`、`-0.1` |
| 互動倍率 `species_action_rate.degree`、`habitat_affect.amount` | **百分比純數字** | `-50`(=×0.5)、`50`(=×1.5) |
| 事件 `effect_value`(reduce_pct) | 百分比整數 | `25`(減 25%) |
| 性狀 `adjust` | 絕對量 | `+2`(食物)、`-1`(AP);`-999` = 完全禁止該行動 |

### 1.6 物種標籤 tags(供 `habitat_affect` 用)

| 物種 | tags |
|---|---|
| red 紅火蟻 | `all,land` |
| green 綠鬣蜥 | `all,land` |
| yellow 貢德氏赤蛙 | `all,aqua,land` |
| brown 美國牛蛙 | `all,aqua,land` |
| blue 領角鴞 | `all,bird` |
| purple 蓋斑鬥魚 | `all,aqua` |

---

## 2. 表格總覽

| # | 分頁名 | 群組 | 用途 | seed 必填? |
|---|---|---|---|---|
| 1 | `students` | 誰在玩 | 學生名單(登入用) | ✅ 必填 |
| 2 | `species_starts` | 誰在玩 | 物種起始棲地與初始隻數 | ✅ 必填 |
| 3 | `species_traits` | 物種基礎值 | 各行動 AP/數量、覓食倍率、食物消耗、標籤 | ✅ 必填 |
| 4 | `traits` | 修正 | 性狀(v5 會生效,10 欄條件式) | ✅ 重點 |
| 5 | `species_action_rate` | 互動 | 同棲地物種互動 → 行動倍率 | ✅ 重點 |
| 6 | `habitat` | 棲地 | 棲地初始 水源/植被/污染/承載量 | ✅ 必填 |
| 7 | `habitat_change` | 棲地 | 物種數量 → 改棲地數值 | ⬜ 可選 |
| 8 | `habitat_affect` | 棲地 | 棲地數值過閾值 → 標籤物種增減 | ⬜ 可選 |
| 9 | `events_pool` | 事件 | 人類事件(固定回合 / 物種棲地閾值) | ✅ 重點 |
| 10 | `scout_pool` | 事件 | 探索卡牌(情報 + 解鎖性狀) | ✅ 可調 |

> **新分頁容錯**:6–8 這 3 張 v5 新表若 Sheets 還沒建,seed 會當作空表略過([seed-from-sheets.js:70-77](../server/scripts/seed-from-sheets.js#L70-L77)),不會整批失敗。但少了它們,棲地數值就不會變動。
> **退役**:`species_prompts` 分頁已不 seed(物種人格改放 [server/src/data/species-personas.json](../server/src/data/species-personas.json));`game_state` 已棄用。

---

## 3. 逐表欄位字典

### 3.1 `students` — 學生名單

| 欄 | 型別 | 必填 | 範例 | 說明 |
|---|---|---|---|---|
| name | 文字 | ✅ | 王小明 | 學生姓名,登入要打對 |
| player_code | 文字 | ✅ | WD-1001 | 玩家代號,登入自動轉大寫;建議 4–10 字英數,**務必唯一**(重複會互搶) |
| species | 文字 | ✅ | red | 物種 id,6 種之一:`red`/`green`/`yellow`/`brown`/`blue`/`purple` |
| team | 文字 | ⬜ | A | 隊伍代號,目前不影響機制,純分組記錄 |

**建議**:30 人,每物種 5 人(對齊 `species_starts.players_count`)。

---

### 3.2 `species_starts` — 物種起始分配

| 欄 | 型別 | 必填 | 範例 | 說明 |
|---|---|---|---|---|
| species | 文字 | ✅ | red | 物種 id |
| home_habitat | 文字 | ✅ | pond | 起始棲地,`wetland`/`forest`/`urban`/`pond` 之一 |
| initial_total | 數字 | ✅ | 120 | 該物種**全體**初始隻數 |
| players_count | 數字 | ✅ | 5 | 該物種有幾位玩家 |

**演算法**:每位玩家分到 = `floor(initial_total / players_count)`,故 `initial_total` 建議能被 `players_count` 整除。

**範例(v5 建議值,5 人制)**:

| species | home_habitat | initial_total | players_count |
|---|---|---|---|
| red | pond | 120 | 5 |
| green | forest | 100 | 5 |
| yellow | wetland | 80 | 5 |
| brown | forest | 60 | 5 |
| blue | forest | 50 | 5 |
| purple | wetland | 70 | 5 |

---

### 3.3 `species_traits` — 物種特性(★ v5 必填)

讓 6 物種真正有差別的核心。**v5 已移除 `forage_power`**,新增 `food_amount`/`food_consume`/`tags`。

| 欄 | 型別 | 必填 | 預設 | 說明 |
|---|---|---|---|---|
| species | 文字 | ✅ | — | 物種 id |
| breed_cost | 數字 | ✅ | 2 | 繁殖一次的 AP 成本 |
| forage_cost | 數字 | ✅ | 2 | 覓食一次的 AP 成本 |
| migrate_cost | 數字 | ✅ | 2 | 遷移一次的 AP 成本 |
| explore_cost | 數字 | ✅ | 1 | 探索一次的 AP 成本 |
| breed_amount | 數字 | ✅ | 5 | 繁殖基數(Step1 base) |
| migrate_amount | 數字 | ✅ | 5 | 遷移一次搬幾隻(無耗損,移出多少到多少) |
| **food_amount** | 數字 | ✅ | 1 | **v5 覓食基礎倍率**;實際食物 = `food_amount × 棲地植被 × N_before` |
| **food_consume** | 數字 | ✅ | 1 | **v5 每隻每回合消耗食物**(0–1) |
| **tags** | 文字 | ✅ | all | **v5 物種標籤**,逗號分隔(見 §1.6),供 `habitat_affect` |
| description | 文字 | ⬜ | — | 老師備註,前端不顯示 |
| initial_trait | 文字 | ⬜ | (空) | 開場就帶的性狀 `trait_id`,可留空 |

> AP 成本下限為 1(性狀調整後最低消耗 1)。

**範例(v5 建議值)**:

| species | breed_cost | forage_cost | migrate_cost | explore_cost | breed_amount | migrate_amount | food_amount | food_consume | tags |
|---|---|---|---|---|---|---|---|---|---|
| red | 2 | 1 | 1 | 1 | 8 | 10 | 1.2 | 0.8 | all,land |
| green | 2 | 1 | 1 | 1 | 20 | 10 | 1.5 | 1.0 | all,land |
| yellow | 2 | 2 | 2 | 1 | 10 | 10 | 0.8 | 1.0 | all,aqua,land |
| brown | 2 | 1 | 1 | 1 | 15 | 10 | 1.3 | 1.2 | all,aqua,land |
| blue | 3 | 2 | 1 | 1 | 8 | 15 | 0.7 | 0.8 | all,bird |
| purple | 2 | 2 | 2 | 1 | 10 | 5 | 0.6 | 0.8 | all,aqua |

---

### 3.4 `traits` — 性狀表(★ v5 大改:會生效,10 欄)

v4 的 traits 只是顯示;**v5 的 traits 會在行動階段與結算實際改數字**。

| # | 欄 | 型別 | 必填 | 合法值 | 說明 |
|---|---|---|---|---|---|
| 1 | trait_id | 文字 | ✅ | 英數-連字號 | 唯一 id,搭配 `scout_pool.unlocked_trait` / `species_traits.initial_trait` |
| 2 | name | 文字 | ✅ | — | 顯示名(中文) |
| 3 | description | 文字 | ⬜ | — | hover 說明 |
| 4 | species | 文字 | ⬜ | `<any>` 或物種 id | 綁定物種;留空 = `<any>` |
| 5 | cond_habitat | 文字 | ⬜ | 棲地 id | 條件棲地;留空 = 不限 |
| 6 | cond_field | 文字 | ⬜ | water/vegetation/pollution | 條件看哪個棲地數值;留空 = 無條件 |
| 7 | cond_threshold | 數字 | ⬜ | 0–1 小數 | 閾值,**正=大於、負=小於**;留空 = 無條件 |
| 8 | action | 文字 | ⬜ | forage/breed/migrate | 作用在哪個行動;留空 = 不作用 |
| 9 | variable | 文字 | ⬜ | food / AP / AM | 變量:`food`=覓食食物、`AP`=該行動點成本、`AM`=該行動數量 |
| 10 | adjust | 數字 | ⬜ | 絕對量 | 調整值;`-999` = **完全禁止該行動**(引擎特殊處理) |

**規則**:
- 欄 5–7 全空 → 無條件,永遠套用。
- 欄 5–10 全空 → 純文字說明,不生效(可放 `scout_pool` 當解鎖線索用)。
- 多條 trait 命中同一行動 → `adjust` 相加(在「①基礎」階段)。

**讀法範例**:`wetland / water / 0.3 / forage / food / +2` = 「在農田、水源 > 0.3 時、覓食食物 +2」。

**範例(v5 建議,選有效果的 21 個)**:

| trait_id | name | species | cond_habitat | cond_field | cond_threshold | action | variable | adjust |
|---|---|---|---|---|---|---|---|---|
| red-pioneer | 拓荒先鋒 | red | | food | -0.2 | forage | AP | -1 |
| red-hitchhike | 搭便車 | red | pond | | | breed | AM | 20 |
| red-network | 族群網絡 | red | | | | migrate | AP | -1 |
| red-egg-attack | 卵攻擊 | red | | | | breed | AM | 0 |
| green-breed | 驚人繁殖 | green | | | | breed | AM | 10 |
| green-amphibious | 水陸兩棲 | green | | | | migrate | AP | -2 |
| green-crop | 啃食農作 | green | | | | forage | food | 1.5 |
| yellow-moist | 濕潤皮膚 | yellow | | water | -0.3 | breed | AP | 1 |
| yellow-ambush | 草叢伏擊 | yellow | | vegetation | 0.6 | forage | AP | -2 |
| yellow-water-dep | 水源依賴 | yellow | | water | -0.4 | breed | AM | -999 |
| brown-breed | 高繁殖 | brown | | | | breed | AM | 5 |
| brown-food-comp | 食物競爭 | brown | | | | forage | AP | 1 |
| blue-tree-nest | 樹洞築巢 | blue | | vegetation | -0.7 | breed | AM | -999 |
| blue-glide | 滑翔翼 | blue | | | | migrate | AP | -2 |
| purple-protect | 育幼保護 | purple | | | | breed | AM | 10 |
| purple-water-bound | 水域限制 | purple | | water | -0.3 | breed | AM | -999 |

> `cond_field` 填 `food` 不是合法棲地數值(合法只有 water/vegetation/pollution),上表 `red-pioneer` 取自草稿、實際請改成有效 metric 或留空。詳見 §5 注意事項。

---

### 3.5 `species_action_rate` — 物種互動倍率(★ v5 新)

同棲地特定物種共存時,改變受影響物種某行動的倍率。

| 欄 | 型別 | 必填 | 合法值 | 說明 |
|---|---|---|---|---|
| affected_species | 文字 | ✅ | 物種 id | 受影響的物種 |
| cond_species_1 | 文字 | ✅ | 物種 id | 條件物種 1 |
| cond_species_2 | 文字 | ⬜ | 物種 id / `X` / 空 | 條件物種 2;`X` 或空 = 不需第二條件 |
| action | 文字 | ✅ | forage/breed/migrate | 受影響行動 |
| degree | 數字 | ✅ | 百分比 | 倍率調整;`-50` = ×0.5、`+50` = ×1.5 |

**規則**:`affected_species` 與條件物種**同棲地**時,該行動 ×(1 + degree/100)。多筆命中 → **相乘**(-50% 與 -30% → ×0.5×0.7=×0.35)。

**範例(節錄,完整見 [v5_data_ready.md §5](v5_data_ready.md))**:

| affected_species | cond_species_1 | cond_species_2 | action | degree |
|---|---|---|---|---|
| yellow | red | | breed | -100 |
| yellow | brown | | breed | -50 |
| red | blue | | forage | -100 |
| green | blue | | forage | -50 |
| brown | brown | | breed | 50 |
| green | green | | breed | 100 |

---

### 3.6 `habitat` — 棲地初始值(★ v5 新,必填)

取代舊版寫死的承載量與前端 resources。

| 欄 | 型別 | 必填 | 合法值 | 說明 |
|---|---|---|---|---|
| habitat | 文字 | ✅ | 棲地 id | wetland/forest/urban/pond |
| water | 數字 | ✅ | 0–1 | 初始水源 |
| vegetation | 數字 | ✅ | 0–1 | 初始植被(覓食食物公式會用到) |
| pollution | 數字 | ✅ | 0–1 | 初始污染(建議先設 0,由事件/habitat_change 動態變) |
| max_capacity | 數字 | ✅ | 整數 | 承載量(超過 → 觸發超載事件) |

**範例(v5 建議值)**:

| habitat | water | vegetation | pollution | max_capacity |
|---|---|---|---|---|
| wetland | 0.9 | 0.6 | 0 | 200 |
| forest | 0.5 | 0.9 | 0 | 400 |
| urban | 0.4 | 0.5 | 0 | 150 |
| pond | 0.85 | 0.5 | 0.2 | 120 |

---

### 3.7 `habitat_change` — 物種改棲地數值(v5 新)

結算 Step6:該棲地該物種總數 ÷ per_count × value → 改該棲地 metric。

| 欄 | 型別 | 必填 | 合法值 | 說明 |
|---|---|---|---|---|
| species | 文字 | ✅ | 物種 id | 哪個物種造成影響 |
| per_count | 數字 | ✅ | 整數 | 每多少隻算一次(預設 100) |
| metric | 文字 | ✅ | water/vegetation/pollution | 影響哪個棲地數值 |
| value | 數字 | ✅ | 0–1 小數(可負) | 變化量,`-0.1` = -10% |

**範例**:

| species | per_count | metric | value |
|---|---|---|---|
| green | 100 | vegetation | -0.10 |
| brown | 100 | water | -0.05 |
| red | 200 | pollution | 0.05 |

---

### 3.8 `habitat_affect` — 棲地數值害/利物種(v5 新)

結算 Step4:棲地某 metric 過閾值 → 對該棲地內帶此 tag 的物種人口 ×(1 + amount/100)。

| 欄 | 型別 | 必填 | 合法值 | 說明 |
|---|---|---|---|---|
| habitat | 文字 | ✅ | 棲地 id 或 `<any>` | 哪個棲地;`<any>` = 全部 |
| metric | 文字 | ✅ | water/vegetation/pollution | 看哪個數值 |
| threshold | 數字 | ✅ | 0–1 小數 | 閾值,**正=大於、負=小於** |
| species_tag | 文字 | ✅ | all/aqua/land/bird | 影響帶此標籤的物種(見 §1.6) |
| amount | 數字 | ✅ | 百分比 | 增減,`-30` = -30%、`-100` = 等於消滅 |

**範例**:

| habitat | metric | threshold | species_tag | amount |
|---|---|---|---|---|
| `<any>` | water | -0.3 | aqua | -10 |
| `<any>` | vegetation | -0.7 | bird | -100 |
| `<any>` | vegetation | 0.6 | aqua | 30 |

---

### 3.9 `events_pool` — 人類事件池(★ 重點)

控制每回合結算觸發的人類事件。v5 新增 `polution` 欄與物種棲地閾值 trigger。

| 欄 | 型別 | 必填 | 合法值 | 說明 |
|---|---|---|---|---|
| event_id | 文字 | ✅ | 英大寫+底線 | 唯一 id |
| name | 文字 | ✅ | — | 中文名 |
| trigger | 文字 | ✅ | 見下 | 觸發條件 |
| target_habitat | 文字 | ✅ | 棲地 id / `<any>` | 影響哪個棲地 |
| target_species | 文字 | ⬜ | 篩選語法(§1.2)**可用標籤** | 影響哪些物種;支援標籤 `aqua`/`land`/`bird`/`all`(僅事件表);留空 = `<any>` |
| effect_type | 文字 | ✅ | reduce_pct / reduce_food | 效果類型 |
| effect_value | 數字 | ✅ | 整數 | reduce_pct=百分比、reduce_food=絕對食物 |
| **polution** | 數字 | ⬜ | 0–1 小數(可負) | **v5 新**:對 target_habitat 的污染增加;0/空=不改。**注意拼字是 `polution`(程式欄名如此)** |
| message | 文字 | ✅ | — | 顯示在生存報告事件區 |

#### trigger 格式

| 格式 | 範例 | 類型 | 意義 |
|---|---|---|---|
| `round_eq:N` | `round_eq:3` | **A** | 第 N 回合觸發 |
| `always` | (固定字串) | **A** | 每回合都觸發 |
| `species_in_habitat_above:N` | `species_in_habitat_above:250` | **B** | 某棲地某物種數 ≥ N(v5 新;搭配該列 target_species + target_habitat 判定) |

#### 事件分兩類:A 類(必定觸發)vs B 類(條件觸發)

事件屬於 A 還是 B,**由 `trigger` 自動決定,不需要也不要另外加欄位**。結算時引擎會替每筆事件標上 `kind`(`A`/`B`),生存報告再據此把事件拆成「A 事件 / B 事件」兩個分頁給學生看。

| 類型 | 對應 trigger | 觸發規則 | 典型用途 |
|---|---|---|---|
| **A 類(必定觸發)** | `round_eq:N`、`always` | **凡命中的全部觸發**,同回合可多筆並存 | 人類開發劇本:排好「第幾回合發生什麼」,時間到一定發生 |
| **B 類(條件觸發)** | 超載(引擎內建)、`species_in_habitat_above:N` | **每回合全域最多 1 筆**;優先序「超載 > 物種棲地閾值」,多筆閾值同時成立時挑「實際數 ÷ 閾值」比例最高者 | 物種失控的負回饋:數量爆掉才懲罰 |

**重點規則**:
- **超載是 B 類的內建事件**(不是 events_pool 的某一列):任一棲地總人口超過 `habitat.max_capacity` 就觸發,效果為該棲地各物種人口 ×(1 − 超出%)。它**優先佔用該回合唯一的 B 類名額**——若本回合有棲地超載,所有 `species_in_habitat_above` 事件都不會觸發。
- 所以老師在 events_pool 只需要管 A 類(`round_eq` / `always`)與 B 類的 `species_in_habitat_above`;超載引擎自動處理,不用填。
- A 類想「每回合都來一點環境壓力」用 `always`;想「劇情排程」用 `round_eq:N`。
- 對應程式:[settlement.js:409-497](../server/src/services/settlement.js#L409-L497)(後端標 kind)、[survival-report.js:227-250](../src/js/components/survival-report.js#L227-L250)(前端 A/B 分頁)。

> `reduce_food` 在 v5「食物各棲地結算」下語意已改變,建議事件改用 `polution`(加污染)/`reduce_pct`(扣人口),更貼近核心主軸「人類破壞棲地 → 棲地惡化 → 物種受害」。

#### effect_type

| 寫法 | 意義 |
|---|---|
| `reduce_pct` | 該棲地 × target_species 的族群 ×(1 − value%) |
| `reduce_food` | 該棲地 × target_species 玩家食物 −value(舊語意,v5 慎用) |

---

### 3.10 `scout_pool` — 探索卡牌(可調)

學生「探索」時隨機抽 1 筆;若 `unlocked_trait` 有值且未解鎖過,一併解鎖該性狀。v4 起含 4 個篩選欄。

| 欄 | 型別 | 必填 | 合法值 | 說明 |
|---|---|---|---|---|
| id | 文字 | ✅ | — | 唯一 id |
| title | 文字 | ✅ | — | 標題 |
| content | 文字 | ✅ | 可含 `\n` | 內容(物種主觀視角) |
| unlocked_trait | 文字 | ⬜ | trait_id | 抽到時解鎖的性狀;留空 = 純情報 |
| available_round | 數字 | ⬜ | 整數 | 第幾回合起可抽;0/空 = 全回合(預設 1) |
| target_species | 文字 | ⬜ | 篩選語法 | 哪些物種能抽到;留空 = `<any>` |
| target_habitat | 文字 | ⬜ | 棲地 id / `<any>` | 哪個棲地能抽到;留空 = `<any>` |
| required_species | 文字 | ⬜ | 篩選語法 | 需同棲地存在某物種才抽得到;留空 = `<any>` |
| required_count | 數字 | ⬜ | 整數 | 搭配 required_species 的門檻隻數(預設 1) |

**範例**:

| id | title | content | unlocked_trait | available_round | target_habitat |
|---|---|---|---|---|---|
| S02 | 同伴不見了 | 昨天在岸邊曬太陽的同伴，今天只剩空殼。 | yellow-moist | 1 | wetland |
| C03 | 破掉的貨櫃 | 一個鐵箱破了洞，爬出很多小小紅色的東西。 | red-egg-attack | 2 | urban |
| P01 | 鐵箱子裡 | 我從鐵箱縫隙爬出來，外面味道完全不一樣。 | red-hitchhike | 1 | pond |

完整 24 張卡見 [v5_data_ready.md §9](v5_data_ready.md)。

---

## 4. events_pool 範例集(複製即用)

> 一場 **5 回合**「人類開發遞進」配置:A 類效果值逐回合加重、污染累積,呼應核心主軸「人類干預才是因」。
> ⚠️ **遊戲為 5 回合**:`round_eq:6` 以上的事件永遠不會觸發。最後一回合(`round_eq:5`)為人類釀成的總崩潰。
> ⚠️ **每個 `round_eq:N` 只放 1 筆 A 事件**:生存報告 A 分頁只完整顯示第一筆,同回合多筆會讓第二筆內容看不到。
> A 事件對全班可見(結算後所有玩家報告都看得到該回合的人類破壞,即使物種未被直接命中)。

| event_id | name | trigger | 類型 | target_habitat | target_species | effect_type | effect_value | polution | message |
|---|---|---|---|---|---|---|---|---|---|
| DEV_FARM_R1 | 農藥噴灑 | round_eq:1 | A | wetland | yellow,purple | reduce_pct | 10 | 0.08 | 農田噴灑農藥，水源變得刺鼻，水裡的生物紛紛逃離。 |
| DEV_SEWAGE_R2 | 家庭廢水 | round_eq:2 | A | wetland | aqua | reduce_pct | 15 | 0.10 | 上游家庭廢水排放增加，水質肉眼可見變濁。 |
| DEV_LOG_R3 | 林木砍伐 | round_eq:3 | A | forest | `<any>` | reduce_pct | 20 | 0.05 | 大片樹木被砍伐運走，失去樹冠遮蔽的棲地變得炙熱乾燥。 |
| DEV_FACTORY_R4 | 工廠廢水 | round_eq:4 | A | wetland | aqua | reduce_pct | 25 | 0.20 | 工廠排放廢水，水源染上異色，水生生物大量死亡。 |
| DEV_COLLAPSE_R5 | 生態崩潰 | round_eq:5 | A | `<any>` | `<any>` | reduce_pct | 35 | 0.20 | 人類的開發讓生態系統全面崩潰，棲地幾乎不適合居住。 |
| THRESH_RED_250 | 咬傷通報 | species_in_habitat_above:250 | B | `<any>` | red | reduce_pct | 20 | 0.1 | 有農夫被紅火蟻咬傷，政府派人處理。 |
| THRESH_GREEN_350 | 植被崩潰 | species_in_habitat_above:350 | B | `<any>` | green | reduce_pct | 30 | 0.1 | 綠鬣蜥太多，森林開始枯死。 |

> 完整事件草稿見 [v5_data_ready.md §8](v5_data_ready.md)(該檔仍是 6 回合版本,搬進 Sheets 時請把 `round_eq:6` 那幾筆併進第 5 回合或捨棄)。

---

## 5. 注意事項與常見坑

1. **改了 Sheets 必須 `npm run seed`** 才會進 Postgres,實驗期間別重跑(會清執行期表,需 `--force`)。詳見 [setup.md §Q1](setup.md)。
2. **`polution` 拼字**:程式欄名就是 `polution`(少一個 l),Sheets 標頭請照打,否則 seed 讀不到、污染永遠 0。
3. **`metric` / `cond_field` 合法值只有 `water` / `vegetation` / `pollution`**。草稿資料([v5_data_ready.md](v5_data_ready.md))出現的 `food` 欄在 v5 不存在,請改對應到 `vegetation` 或留空,否則該列靜默失效。
4. **閾值方向**:正=大於、負=小於、等於不套用。寫反了效果會相反。
5. **`-999` 是禁止標記**:`traits.adjust` 填 `-999` 代表完全禁止該行動,別當成「減 999」。
6. **trigger 拼錯靜默失敗**:`round_eq=1`(應為 `round_eq:1`)不會報錯也不觸發,seed log 可留意筆數。
7. **新增第 7 種物種**:物種清單 hardcode 在前後端,只改 Sheet 不夠,需改 [src/js/data.js](../src/js/data.js) 與 [server/src/services/filter-parser.js](../server/src/services/filter-parser.js)。

---

## 附:相關檔案

- 部署 / 啟動 / FAQ:[docs/setup.md](setup.md)
- v5 機制設計與結算公式:[docs/mechanic_v5_plan.md](mechanic_v5_plan.md)
- 可直接填入的完整資料集:[docs/v5_data_ready.md](v5_data_ready.md)
- 欄位真相來源:[server/src/db/schema.sql](../server/src/db/schema.sql) / [server/scripts/seed-from-sheets.js](../server/scripts/seed-from-sheets.js)
- 結算引擎:[server/src/services/settlement.js](../server/src/services/settlement.js)
</content>
</invoke>
