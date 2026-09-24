# FIRE Calculator 改進計劃：Vanguard 動態提領與歷史區間對照

## 目標

把目前的 FIRE calculator 從「提領率 + 回測表」升級成「退休支出決策工具」。新版沿用已確認的 layout：左側集中輸入，右側呈現結果、年度推演與歷史區間對照。

核心新增能力：
- `Vanguard 動態提領` 作為獨立策略
- `最糟區間 / 中位區間 / 最好區間` 三種真實歷史 rolling window 對照
- 讓使用者看見支出彈性成本，而不是只看成功率
- 明確切分「可浮動生活提領」與「貸款支付」的模型語意

## 產品原則

1. 不做新 app
   繼續維持單頁、browser-local、GitHub Pages 可部署。

2. 不把 Vanguard 動態提領混成護欄提領
   Vanguard dynamic spending 是「比例提領 + 上下限限制年度支出變動」。現有護欄比較像依提款率偏離程度調整支出，兩者要分開。

3. 不用平均報酬路徑當代表
   `平均區間` 容易抹掉 sequence risk。改用 `中位區間`：從真實歷史 rolling windows 中找最接近中位結果的一段。

4. 回測不是預測
   UI 文案要清楚說明：歷史區間是壓力測試與參考，不是保證。

5. 貸款支付不是 Vanguard 可調整的生活費
   房貸與其他貸款是名目固定的攤還現金流，不應被通膨調整，也不應被 `+5% / -2.5%` 的年度支出限制裁切。這不是單純命名問題：目前 `nextBacktestWithdrawal` 會把 `previousInflation` 套到整筆 withdrawal；若把貸款放進 withdrawal，1970s 這類高通膨窗口會系統性高估貸款支出。

## Phase 2 前置裁定：回測模式與帶債模式定義

這是進入 Vanguard + 貸款拆分實作前的 blocker。

目前 FIRE 目標是：

```text
initialFireNumber = annualExpenses / safeWithdrawalRate
```

其中 `annualExpenses` 是生活支出，不含貸款。累積期則另外用 `debtPayment` 扣現金流；`無債 FIRE` 代表「負債歸零且投資資產達標」。所以目前模型預設語意是：回測起點在無債 FIRE 之後。

產品裁定採雙模式（Carter 2026-09-24 確認要支援「還有房貸就退休」）：

1. `無債 FIRE 回測`（預設）
   - 回測中貸款支付恆為 0。
   - UI 可顯示貸款如何影響達成 FIRE 的時間，但不把貸款帶進退休回測。
   - Vanguard 實作較單純，但「貸款支付拆分」只保留為模型安全界線。

2. `帶債退休回測`（明確 opt-in）
   - 新增輸入：`退休起始年（距今幾年）`。
   - 不讓使用者手填 `退休起始時貸款剩餘年限`。剩餘貸款狀態要從現有 projection 快照推導，避免跟今天的 `mortgageBalance` / `loanBalance` 產生矛盾。
   - `退休起始年` 一旦成為輸入，所有錨定在退休時點的量都要來自同一個時間點，不可混用今天的 `inputs.*` 與第 N 年 projection。
   - 帶債模式的預設回測起始資產與首年生活提領：

```text
若退休起始年 N > 0：
  row = projection 中 year = N 的那一列
  回測起始資產 = row.fireNumber + row.mortgageDebt + row.loanDebt
  首年生活提領 = row.expensesAnnual

若退休起始年 N = 0：
  回測起始資產 = initialFireNumber + 今天的貸款餘額
  首年生活提領 = inputs.annualExpenses
```

   - 因為 projection 第一次 push 是第 1 年，沒有第 0 年 row；實作時只用 `row.year` 對齊退休起始年，並把 N 夾擠在可用 projection 範圍內。不要用 `N - 1` 索引假設，因為 projection 可能包含非整年尾列。
   - 若使用者手動填 `回測起始資產`，UI 要標示這是覆寫值，可能不含償債準備。
   - 回測要追蹤 `debtShortfallYear`，並與生活提領不足的 `failureYear` 分開。
   - 每個回測年度扣款順序固定為：先扣貸款支付，再扣生活提領。貸款扣不出來記 `debtShortfallYear`；貸款後剩餘資產不足生活提領才記 `failureYear`。
   - 若 `退休起始年 >= debtFreeYear`，顯示「此設定下退休時已無債，結果與無債模式相同」。

雙模式的目的不是把 C blocker 藏到 toggle 後面，而是保住既有預設語意，同時讓帶債模式有自己的起始資產、失敗語意與 UI 標籤。實作可以開始，但必須遵守上述帶債模式定義。

## Layout 改進

### 左側輸入區

新增或重組成以下區塊：

- `現金流`
- `目前資產`
- `貸款`
- `假設值`
- `新增投資配置`
- `退休提領`

`退休提領` 區塊放在左側底部，包含：

- `退休提領策略`
  - `通膨定額`
  - `Vanguard 動態`
  - `護欄`
  - `固定比例`
- `初始提領率`
- `提領年限`
- `回測起始資產`
- `回測模式`
  - `無債 FIRE 回測`
  - `帶債退休回測`
- 帶債模式才顯示：`退休起始年（距今幾年）`

當策略選 `Vanguard 動態` 時顯示：

- `年度生活提領最多增加 +5%`
- `年度生活提領最多減少 -2.5%`
- 衍生顯示：`實際初始提領率`，由 `首年生活提領 / 回測起始資產` 算出

當策略選 `護欄` 時顯示：

- `護欄觸發幅度`
- `護欄調整幅度`

### 右側結果區

保留既有摘要：

- `FIRE 目標`
- `投資資產達標`
- `無債 FIRE`
- `Coast FIRE`
- `目前儲蓄率`

新增主要區塊：

- `歷史區間對照`
  - `最糟區間`
  - `中位區間`
  - `最好區間`

成功率標籤依模式切換：

- 無債模式：`成功率（無債 FIRE）`
- 帶債模式：`成功率（含償債）`

帶債模式要分開顯示兩類失敗：

- `生活提領失敗`
- `貸款支付短缺`

每張區間卡顯示：

- 起始年份與結束年份
- 是否耗盡
- 若允許帶債退休：貸款是否繳得出
- 最低年度提領
- 最大年度降幅
- 低於初始支出的年數
- 總提領
- 期末資產

## Vanguard 動態提領模型

建議公式：

```text
可支應生活資產 = 年初或前一年年底投資組合 - 剩餘貸款餘額

生活提領率 = 首年生活提領 / 可支應生活資產

候選提領 = 可支應生活資產 × 生活提領率

通膨調整基準 = 去年實際提領 × (1 + 前一年通膨率)

今年生活提領 = clamp(
  候選提領,
  通膨調整基準 × (1 + 下限百分比),
  通膨調整基準 × (1 + 上限百分比)
)
```

預設值：

- 年度生活提領最多增加：`+5%`
- 年度生活提領最多減少：`-2.5%`
- Vanguard 候選提領使用衍生的生活提領率：`firstLifestyleWithdrawal / (startingAssets - startingDebt)`，不是 `firstWithdrawal / startingAssets`。帶債模式的 `startingAssets` 含償債準備，若拿它當分母會稀釋生活提領率；償債準備金只能保障貸款支付，不應永久壓低生活提領候選值。

注意：
- 下限在資料模型中統一存 signed fraction，例如 `-0.025`。計算前驗證 `-1 < floor <= 0`、`ceiling >= 0`、`ceiling >= floor`，不要靜默修正。這可以同時擋住 `0.025` 被誤解成「每年至少加薪」以及 `-2.5` 讓 floor 等於不存在。
- clamp 只套在 `生活提領`，不套在貸款支付。
- `Vanguard 動態` 與 `護欄` 的比率判斷都只看 `portfolio - remainingDebt` 的生活資產部位，避免被尚未支出的償債準備金稀釋。
- 若走「允許帶債退休」，當年總現金流為：

```text
年度總提領 = 生活提領（動態） + 貸款支付（名目固定，攤還至結束）
```

貸款支付應以攤還表逐月計算後彙整成年度金額；不可直接用 `payment * 12` 近似到與累積期不一致。

## 貸款與圖表語意

避免在回測區引入另一個含義不同的「總支出」。現有輸入的 `年度支出` 是生活費，tooltip 也已另列 `年度貸款支出`。回測圖表改用：

- `生活提領（動態）`
- `貸款支付（名目固定）`
- `年度總提領`

若允許帶債退休，圖表用堆疊呈現：貸款支付在下、生活提領在上，並標出無債年份。另加一條 `通膨調整基準` 虛線，讓使用者看見 Vanguard ceiling/floor 何時真的生效。投資組合餘額不要與年度提領共用同一個 Y 軸。

兩個模式下的 `最糟 / 中位 / 最好` 可能不是同一段歷史區間。模式切換後若代表區間起始年份改變，卡片要有視覺提示，避免使用者以為只是在同一區間上多加貸款。

## 歷史區間選取邏輯

每個 rolling window 都計算：

- `success`
- `failureYear`
- `debtShortfallYear`（若允許帶債退休）
- `endingPortfolio`
- `totalWithdrawal`
- `minAnnualWithdrawal`
- `maxSpendingCut`
- `yearsBelowInitialWithdrawal`
- `maxDrawdown`

排序建議：

1. `最糟區間`
   - 若允許帶債退休，先看是否出現 `debtShortfallYear`
   - 先看是否失敗
   - 再看最低年度提領與最大支出降幅
   - 最後看期末資產

2. `中位區間`
   - 用綜合分數排序後取中位數
   - 不使用平均報酬合成路徑

3. `最好區間`
   - 成功且支出壓力低
   - 期末資產高

## 實作階段

### Phase 1：UI 結構整理

- 把現在的 withdrawal/backtest inputs 整理到 `退休提領` 區塊
- 用 strategy selector 控制欄位顯示
- 保留現有輸入值與預設情境

驗收：
- 切換策略時只顯示相關欄位
- 手機版不爆版
- 現有 FIRE/Coast/敏感度不退化

### Phase 2：新增 Vanguard 策略

進入條件：
- 已由 Carter 確認採用雙模式：預設 `無債 FIRE 回測`，另開 `帶債退休回測`
- 帶債模式已定義 `退休起始年（距今幾年）`，並從同一個 projection 年度 row 取得 `fireNumber`、`expensesAnnual`、`mortgageDebt`、`loanDebt`
- 帶債模式已定義預設起始資產為 `row.fireNumber + row.mortgageDebt + row.loanDebt`，N = 0 時才使用今天的 `initialFireNumber + initialDebt`
- 帶債模式已定義 `退休起始年 = 0` 與 N 超出 projection 長度時的邊界處理
- 帶債模式已定義 `退休起始年 >= debtFreeYear` 的退化提示
- 已定義 Vanguard 的 `initialRate` 來源
- 已定義 ceiling/floor 的 signed fraction schema

- 新增 `vanguardDynamic` strategy key
- 實作 annual withdrawal bounded by ceiling/floor
- 僅對 `生活提領` 套 Vanguard 規則
- 加入固定測試情境做 sanity check

驗收：
- 上限/下限調整會改變年度提領結果
- 0% 上限/下限時支出幾乎固定
- 固定比例、通膨定額、護欄策略不受影響
- 貸款支付不會被通膨調整，也不會被 Vanguard ceiling/floor 調整
- 若允許帶債退休，每個 rolling window 都重新建立貸款狀態，不共享 mutated debt object
- 若允許帶債退休，每個回測年度先扣貸款、再扣生活提領；不得把貸款違約誤歸類成生活提領失敗
- 帶債模式成功率標籤與無債模式不同，且分開呈現生活提領失敗與貸款支付短缺
- Phase 2 要補一個高通膨窗口實測對照，驗證貸款不再被通膨化後的總提領差異

### Phase 3：歷史區間對照

- 將 rolling window 結果擴充 lifestyle metrics
- 產生 best/median/worst 三張卡
- 加入小型折線圖或 compact table

驗收：
- 三個區間來自真實起始年份
- 最糟不是只用最低期末資產，而是能反映支出痛感
- 若允許帶債退休，`debtShortfallYear` 會優先反映在最糟排序與卡片徽章
- 若模式切換造成代表區間年份改變，卡片有清楚提示
- JSON export 包含三個代表區間

### Phase 4：文案與文件

- README 補 Vanguard dynamic spending 說明
- UI 補一句「歷史區間不是預測」
- JSON export 補 strategy settings 與 representative windows

驗收：
- 使用者能從 UI 理解每個欄位影響什麼
- 匯出資料足夠重建回測摘要

### Phase 5：驗證

- `node --check` extracted script
- Chromium DOM dump：預設頁面有代表區間
- Strategy smoke tests：
  - 通膨定額
  - Vanguard 動態
  - 護欄
  - 固定比例
- Desktop/mobile screenshots

## 建議優先順序

先做 `Phase 1` 與 Phase 2 前置裁定。
原因是 layout 與 Vanguard 策略本身是產品主軸，但帶債退休會改變 FIRE 目標、回測起始資產、成功率標籤與失敗語意，不能在未裁定時直接動回測核心。

裁定後再做 `Phase 2`，接著做 `Phase 3`，把歷史區間變成使用者真正能比較的決策畫面。

最後再做 README、JSON export、視覺 polish。
