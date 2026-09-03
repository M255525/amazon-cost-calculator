# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

**Amazon 跨境電商成本分析計算機**，單檔前端、無後端、無序號授權。來源是使用者提供的參考檔 `C:\Users\mark_\SynologyDrive\簡報資料\2023\輔仁大學\h2\參考資料\AMAZON-3-成本分析表.xlsx`（「Amazon平台賣價推估售價對應成本計算表」，分 FBM型／FBA型 兩張工作表，各自平行計算 USD／JPY 欄位），2026-09-03 依使用者要求做成互動網站，型態仿 `資料儀表板/restaurant-feasibility-calculator`（即時試算＋健康帶判色＋內建範例＋BYOK AI診斷）。

**目前僅為本機工具，未上線 GitHub Pages、未建公開 repo**（來源 Excel 標註「資料來源：展貿科技有限公司」屬於真實公司的營運參數，只沿用其計算方法論，內建 5 組範例已全部改為虛構品類；是否要對外公開待日後另行決定）。

## 與來源 Excel 的關係與差異

- 保留原始的成本瀑布結構：期望售價 → 扣平台手續費 → 扣FBA費用 → 扣當地稅務/雜費 → 扣退貨風險 → 平台淨利 → 扣金流手續費 → 實收金額 → 換算台幣 → 扣頭程運費 → 依安全margin算出建議採購價格區間（最高/最低）。
- **公式修正**：原始 Excel 有處不一致——USD 欄「當地稅務/雜費」用上一步淨額（FBA費用扣完後）計算，但 JPY 欄同一列卻用最原始售價計算（判斷為複製公式時的疏漏）。本工具**統一改用「上一步淨額」逐層扣的 cascade 算法**（`calcMarket()` 的 `localFeeAmt = netB * localFeeRate`），USD/JPY 兩欄邏輯一致。「退貨風險」列則沿用原始 Excel 兩欄本來就一致的算法（基於最原始售價）。
- 開發期已用原始 Excel FBM表 USD欄（該欄本來就是 cascade 算法）的真實中間值反推驗證 `calcMarket()` 的正確性（price=399.99, platformFeeRate=0.25, fbaFee=0, localFeeRate=0.1, returnRiskRate=0.05, paymentFeeRate=0.003, exchangeRate=27, freight=1191, margin=15%/30% → 逐格與 Excel F8~F20 完全吻合），已用 Node 腳本跑過，數字未進最終 UI（只是內部正確性檢查）。
- 出貨模式（FBM/FBA）在本工具中對兩個市場（USD/JPY）統一套用同一個全域切換，跟原始 Excel「兩張分開的工作表」不同（原始 Excel 的 JPY 欄 FBA費用在兩張表都固定套用，屬於該公司資料本身的不一致，本工具改為讓 FBM 模式同時清零兩個市場的 FBA費用，邏輯更一致）。

## 架構

`index.html` 單一 IIFE `<script>`，型態逐字比照 `資料儀表板/restaurant-feasibility-calculator/index.html` 已驗證的模式（VAR_DEFS驅動表單＋單一計算源＋扇出渲染＋BYOK AI診斷整包），細節見計畫檔 `~/.claude/plans/c-users-mark-synologydrive-2023-h2-amaz-ticklish-nautilus.md`（若已被清除則以下列各函式本體為準）：

- `VAR_DEFS` — 變數定義陣列，`group` 分三類：`global`（安全margin，兩個市場共用）／`usd`／`jpy`。`buildGrids()` 依 group 分別塞進 `#globalGrid`／`#varGridUSD`／`#varGridJPY` 三個容器（不像 restaurant 版有 tab 切換，因為此工具刻意讓美日兩站並排同時可見，對應 Excel 原始的雙欄並列結構）。
- `state.fulfillmentMode`（`"FBA"`/`"FBM"`）— 獨立於 VAR_DEFS 之外的全域切換，`buildModeUI()`／`syncModeUI()` 管理；FBM 模式下 `calcMarket()` 會把兩個市場的 `fbaFeeApplied` 都清零，並在 FBA費用滑桿下方顯示提示文字（`updateFbaNotes()`）。
- `PRESETS` — 5 組虛構品類範例（居家收納／3C配件／戶外露營／寵物用品／美妝保養），`applyPreset()` 覆寫 `state` 後重算，比照 restaurant 版的 `Object.assign` 風格。
- `calcMarket(price, platformFeeRate, fbaFee, localFeeRate, returnRiskRate, paymentFeeRate, exchangeRate, freightPerUnitTWD, highMarginPct, lowMarginPct, fulfillmentMode)` — 核心計算引擎，USD/JPY 各呼叫一次；`calculate()` 回傳 `{usd, jpy}`。
- `renderMarket(prefix, currencyLabel, res)` — 把單一市場的計算結果畫成瀑布式帳本（`.ledger` / `ledger-row`）＋最高/最低採購價格卡片＋淨利率徽章（`marginState()` 三段判色：≥20% good／10-20% warn／<10% bad）＋警示訊息（`afterFreightTWD<=0` 或 `minPurchasePriceTWD<=0` 時顯示紅色警告）。
- 頭程運費小工具（`bindFreightTool()`）— 每個市場一組「總運費(TWD) ÷ 件數」輸入框＋按鈕，計算後直接寫回對應的 `freightPerUnitUSD`/`freightPerUnitJPY` 滑桿與 state，對應 Excel 的 I18/K18 輔助算法；這兩組輸入框本身不存進 `state`/localStorage（純計算小工具，不是正式變數）。
- `AI_PROVIDERS`／`callLLM`／BYOK 設定面板 — **逐字比照** restaurant-feasibility-calculator 已驗證的實作（Claude 需要 `anthropic-dangerous-direct-browser-access` header；OpenAI/Gemini/OpenRouter 無此限制；429/500/503/529 自動重試3次）。金鑰只存 localStorage（`amazonCostCalcApiConfig`），不經任何後端。
- `buildDiagnosisPrompt(res, extra)` — 把兩個市場的售價/淨利率/採購價格區間整理成 prompt；`AI_PROMPT_PRESETS` 5組快速角度（定價健康度／FBM或FBA划算／頭程運費侵蝕／退貨風險保守估算／美日兩站優先順序）。
- `ruleBasedDiagnosis(res)` — 無金鑰或AI失敗時的規則式 fallback，依兩個市場的淨利率分級與採購空間是否為負分別給建議句。

### localStorage

- `amazonCostCalcState` — 目前所有變數值＋`fulfillmentMode`（reload 後還原）。
- `amazonCostCalcApiConfig` — `{provider, model, apiKey, extra}`，只存本機瀏覽器。
- `amazonCostCalcActivePreset` — 目前選取中的品類 preset id。

「重設為基準假設」（`resetToDefaults()`）只重設 `VAR_DEFS`／`fulfillmentMode`／`activePresetId` 並改寫 `amazonCostCalcState`，不動 `amazonCostCalcApiConfig`——AI 設定不隨重設清除，與 restaurant 版慣例一致。

## 本次範圍刻意不做的功能

比照使用者明確決定的範圍（先做核心試算＋AI診斷），以下功能**目前沒有**，之後有需要再個別評估要不要加：PDF匯出、頂部跑馬燈、`manual.html` 操作手冊、PWA加入主畫面、訪客計數器、互動平面圖類的 signature 視覺元素、可攜式桌面版 exe。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）或用伺服器託管即可。

預覽伺服器：port `8809`（工作區根目錄 `.claude/launch.json` 的 `amazon-cost-calculator` 項目），用 Preview MCP 的 `preview_start` 啟動；若該 MCP 在當次工作階段不可用，退回 `python -m http.server 8809 --directory 資料儀表板/amazon-cost-calculator` 暫起、測完關閉（開發時已用此方式＋Playwright 驗證過核心互動：出貨模式切換、5組範例套用、頭程運費小工具、警示訊息觸發、規則式AI診斷、重設按鈕、手機寬度無橫向溢出）。
