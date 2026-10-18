# 项目交接文档 · Free Invoice Generator（免費請求書產生器）

> 文档生成日期：2026-09-13（周日）
> 适用对象：接手本项目维护 / 二次开发的人员
> 仓库：https://github.com/takayashoshi888/free-invoice-generator （分支 `main`，公开）

---

## 1. 项目概述

- **名称**：Free Invoice Generator（免費請求書產生器）
- **定位**：免费、纯前端、无需注册的日式请求书（发票）生成器
- **核心流程**：填写资料 → 上传印章（公司角印 / 个人印鉴）→ 一键导出 PDF 或打印
- **数据隐私**：所有数据仅存于浏览器 `localStorage`，**无后端、无账号、无数据上传**
- **目标市场**：日本（UI 提供 中文 / 日本語 双语切换）

---

## 2. 技术架构

| 项目 | 说明 |
|---|---|
| 形态 | 纯静态单文件 HTML + 原生 JavaScript |
| 构建 | 无构建步骤、无包依赖、无后端 |
| 第三方 CDN | `html2canvas`、`jsPDF`（PDF 导出）、`Noto Sans JP`（字体） |
| 持久化 | `localStorage`（草稿、语言偏好） |
| 双语方案 | 所有文案集中在各页 `<script>` 内的 `I18N = { zh, ja }` 对象，由 `applyLang()` 通过 `[data-i18n]` 的 `textContent` 覆写 |
| 主題方案 | 顏色全部收斂為 CSS 變數；切換 `<html data-theme="...">` 即換版。偏好存 `localStorage.fig_theme`（首頁與應用頁**共用**），`<head>` 內有一段同步腳本在首次繪製前套用，避免白閃。**應用頁採雙命名空間**：`--doc-*`（A4 本體，永不隨主題變動）與 `--ui-*`（介面，隨主題變動），見 §4d |
| 明細表頭顏色 | 文件級設定，存於草稿 `state.barTheme`（`classic` / `blue` / `black` / `gray` / `orange`）。以 `#invoicePage[data-bar="..."]` 覆寫 `--doc-header-bg`，**會影響 PDF**，與介面主題正交，見 §4e |

> ⚠️ **改文案必读**：新增带文字的按钮 / 提示，必须同时补 `data-i18n` 属性与 `zh`、`ja` 两个字典条目，否则对应语言会漏译或显示键名。

---

## 3. 文件结构

```
/
├── index.html        # 落地页（品牌首页、功能介绍、使用说明）
├── seikyusho.html    # 应用页（表单 + A4 预览 + PDF 导出）
├── logo/             # 品牌图片资源（金线鸟居 logo.png）
├── HANDOFF.md        # 本交接文档
├── README.md         # 项目说明文档
└── .gitignore        # 排除本地/工具/部署文件
```

> 注：`project.config.json`、`project.private.config.json`（微信开发者工具配置）、
> `.edgeone/`（EdgeOne 部署产物）、`.workbuddy-ai/`（AI 工具元数据）、`.env` 均**不入库**。

---

## 4. 今日变更记录（2026-09-13）

### 4.1 印章生成器入口（commit `58be2e6`）

- `seikyusho.html`：在功能栏（`.toolbar`）「← 返回主頁」之后新增「🔏 印章產生器」按钮；
  并在印章上传区（個人印章文字栏下方）新增情境提示框。两者均跳转
  `https://sealkit.app/kaishain`（`target="_blank"` + `rel="noopener noreferrer"`）。
- `index.html`：导览列（`.nav-actions`）「使用說明」之后新增同入口。
- i18n 新增：`toolbar_sealgen` / `sealgen_hint` / `sealgen_hint_link` / `nav_sealgen`（zh + ja）。
- README：功能特色新增条目，使用方法新增第 6 步。
- 顺带修复：原「使用說明」按钮缺失 `data-i18n="nav_help"`，导致日语模式不翻译（字典已存在，仅缺属性）。
- 新增 `.gitignore`：排除 `.env`、`.workbuddy-ai/`、`.edgeone/`、`project.config.json`、`project.private.config.json` 及 OS/编辑器杂讯。

### 4.2 Cloudflare Web Analytics（commit `d26708d`）

- `index.html` 与 `seikyusho.html` 均在 `</head>` 前插入 Web Analytics beacon：
  ```html
  <!-- Cloudflare Web Analytics -->
  <script type='module' src='https://static.cloudflareinsights.com/beacon.min.js'
          data-cf-beacon='{"token": "b9aa7b35f6d3470e957fb6ad7c77b7e5"}'></script>
  <!-- End Cloudflare Web Analytics -->
  ```
- token：`b9aa7b35f6d3470e957fb6ad7c77b7e5`（刻意写入客户端，符合 Cloudflare Web Analytics 设计，**非密钥**）。
- ⚠️ **待观察**：本写法用 `type='module'`（用户提供），官方默认是 `defer`。若 Cloudflare 控制台 24h 内仍无数据，先将 `type='module'` 改为 `defer` 再推送。

---

## 4b. 变更记录（2026-09-30）

### 4b.1 修复：行动端（手机）PDF 只导出一半内容（commit `20a44fd`）

**症状**：电脑端导出 PDF 正常完整，手机端（如 iPhone）导出的 PDF 只显示约一半内容 —— 后半段（部分明细、小计／消费税／合计、振込先、页脚）完全消失，且全程无任何报错。

**根因（已量化）**：`downloadPDF()` 固定把整张 canvas 以「宽 210mm」贴到**单一张 A4**，图片高度由 canvas 长宽比推导（`imgH = canvas.height × 210 / canvas.width`）。而 `fitInvoice()` 会在容器宽 < 794px 时给 `#invoicePage` 加上 `.is-mobile`，让预览回流成「宽 100%、内容纵向堆叠」的窄长版面：

| 设备 | 预览元素 | canvas @2x | 推导图高 | 单页可见 |
|---|---|---|---|---|
| 桌面 1440px | 794×1123（A4 比例 0.707） | 1588×2246 | 297.0mm | 100% |
| iPhone-13 390px | 370×1082（比例 0.35） | 740×2164 | 614.1mm | **48.4%** |
| iPhone-SE 375px | 355×1017 | 710×2036 | 602.2mm | 49.3% |

桌面之所以正常，是因为 794:1123 恰好等于 A4 比例（297.0mm）；手机版面被拉长约 2 倍 A4 高，超出 297mm 的下半部被画到页面框外，PDF 检视器直接裁掉。

**修法**：
- 导出改用「A4 桌面版」离线副本（`buildExportNode()`：克隆 `#invoicePage`、移除 `.is-mobile`、锁 794px、放进 `position:fixed; left:-10000px` 的 stage 截图）→ 输出比例恒为 A4，手机与电脑结果完全一致。
- `composePdf()` 三分支：≤297mm 单页 / ≤297×1.12 等比微缩单页 / 超过则逐页 `addImage(y=-offset)` 分页 —— 任何情况都不裁切内容。
- iOS Safari 加固：等待 `document.fonts.ready` 与图片解码、`pickExportScale()` 依 16.7M px canvas 面积上限降倍率、`canvasHasInk()` 空白画布守卫、`pdfBusy` 连点防护、`flash(msg, ms)` 支持自定义时长。
- 不再修改线上 `#invoicePage`（旧代码会临时摘除 `paper-shadow`），预览不再闪动。

> ⚠️ **改 PDF 导出必读**：必须走 `buildExportNode()` 产出的 A4 离线副本，**绝不可**直接对线上（可能已被 `.is-mobile` 回流的）`#invoicePage` 截图，否则本缺陷会立即重现。

**验证方式**：Chromium + `isMobile` 视口 + pdf.js 回渲目视核对。手机导出 794×1123 → 297.0mm → 1 页、MediaBox `595.28×841.89pt`（= A4）无裁切；25 条明细 2 页、60 条明细 + 长地址 3 页且分页位移精确等于 297mm；含上传 Logo／角印／个人印 1 页；连点 5 次仅生成 1 次；桌面 1588×2246 与修正前一致（无回归）。

---

## 4c. 變更記錄（2026-10-18）

### 4c.1 首頁新增六版主題切換

**需求**：首頁提供多版本配色，含「原版 + A 黑白正式 / B 藍色商務 / C 灰色極簡 / D 黑金品牌 / E 橘黃」。

**實作**：
- `index.html` 的 `<style>` 重構為**全變數化**：原 `--navy / --navy-2 / --cyan / --violet` 全部移除，改為語意令牌（`--bg / --card / --heading / --ink-2 / --muted / --muted-2 / --line / --line-soft / --accent / --accent-ink / --grad / --btn-primary-grad / --on-primary / --final-* / --shadow-* / --blob-* / --seal / --sealgen-* / --overlay` 等 40+ 項）。
- 六版定義：`:root`（原版）+ `html[data-theme="mono|blue|gray|gold|orange"]`。`gold` 為深色版並帶 `color-scheme: dark`。
- 切換 UI：導覽列 🎨 下拉選單，6 項各附色票；`setTheme()` 寫入 `localStorage.fig_theme` 並同步 `<meta name="theme-color">`。
- 無白閃：`<head>` 內同步腳本在首次繪製前讀 `fig_theme` 並設 `data-theme`（實測 `DOMContentLoaded` 時屬性已就位）。
- 過渡動畫：切換時掛 `html.theme-switching` 320ms，僅在此期間套用顏色 transition，不干擾常態 hover；並以 `prefers-reduced-motion` 關閉。
- i18n：新增 `theme_label / theme_classic / theme_mono / theme_blue / theme_gray / theme_gold / theme_orange`（zh + ja），`applyLang()` 同時刷新觸發按鈕文案。

**順帶修復（原版既有缺陷）**：
- `.section h2 { color: var(--heading) }` 的優先級高於 `.final` 的繼承色，導致 **CTA 區標題在深色底上幾乎不可見**（實測對比度僅 1.32:1）。已補 `.final h2 { color: var(--final-ink) }`，現為 8.94:1。
- `--muted-2`（頁尾、Hero 迷你說明）原為 2.3–2.9:1，不符 WCAG AA；六版皆已調整至 ≥4.5:1。
- 區塊眉標原用 `--accent`（青色 2.28:1）；新增 `--accent-ink` 專供文字使用（classic `#0e7490`、orange `#c2410c`），保留 `--accent` 給漸層／裝飾。

**約定（新增主題必改 6 處）**：`:root` 或 `html[data-theme]` 令牌區塊 → `THEMES` 陣列 → `THEME_META`（含 `theme-color`）→ `I18N.zh` / `I18N.ja` 文案 → `.sw-xxx` 色票 → 選單 `<button>`。漏一處會出現「選單有項但無法切換」或「顯示鍵名」。

**驗證方式**：Chromium 逐版截圖（1440×900）＋ 以 getComputedStyle 實測 15 組文字／背景組合的 WCAG 對比度（含漸層取最差 stop、rgba 逐層合成），6 版 × 15 項 = 90 項**全部 ≥ AA**；另驗持久化、首帧屬性、選單開合與外部點擊關閉、中日文聯動、390px 行動端（導覽列 2 行、無橫向溢出）。

**範圍限制**：本輪主題僅套用於 `index.html`。`seikyusho.html` 於同日稍後另立令牌完成（見 §4d），**未沿用首頁令牌**，以避免影響已修復的 PDF 版面。

---

## 4d. 變更記錄（2026-10-18）· 應用頁主題切換

### 4d.1 需求與拍板結論

用戶要求為 `seikyusho.html` 加上主題切換，清單為 **5 版**（原版 / 黑白正式 / 藍色商務 / 灰色極簡 / 橘黃），**刻意不含「黑金品牌」**。三項決策已向用戶確認：

| 問題 | 用戶決定 |
|---|---|
| 主題是否影響 A4 請求書本體（PDF 那張紙）？ | **只換介面** — 請求書本體永遠白紙黑字，保證列印與正式送出可用 |
| 本頁要不要也加「黑金品牌」？ | **不加，本頁 5 版** |
| 主題偏好是否與首頁共用？ | **共用同一份 `fig_theme`** |

### 4d.2 核心架構決策：雙令牌命名空間

這是本次唯一真正重要的設計。若「介面」與「紙張」共用同一組變數，任何一次主題覆寫都可能悄悄改到 PDF 輸出 —— 而且**不會有任何報錯**，只會在某天發現存檔的請求書表頭變色了。因此把變數**在命名上就切開**：

```css
:root {
  /* A4 請求書本體 —— 固定，任何 html[data-theme] 都不得覆寫 */
  --doc-ink: #111;
  --doc-line: #444;
  --doc-header-bg: #1e3a5f;
  --doc-gray-bg: #f5f5f5;
  --doc-bank-bg: #f0f0f0;
  --doc-seal: #c0392b;   /* 印章紅 */

  /* 介面 —— 預設 = classic（原版深藍），由各主題區塊覆寫 */
  --ui-bg / --ui-card / --ui-card-2 / --ui-card-3 / --ui-line / --ui-line-2
  --ui-ink / --ui-ink-2 / --ui-muted / --ui-muted-2 / --ui-heading
  --ui-primary / --ui-primary-hover / --ui-on-primary / --ui-accent-ink
  --ui-field-bg / --ui-field-border / --ui-focus / --ui-focus-ring
  --ui-hover / --ui-hover-2 / --ui-active-bg / --ui-active-ink
  --ui-danger / --ui-danger-hover / --ui-dash
  --ui-sealgen-bg / --ui-sealgen-ink / --ui-sealgen-line / --ui-sealgen-hover
  --ui-overlay / --ui-shadow-menu / --ui-shadow-modal / --ui-shadow-paper
}
```

> ⚠️ **硬規則（改樣式前必讀）**
> - 要改 **A4 紙張本體** → 只能動 `--doc-*`（或紙張專屬的 `.invoice-page` 內樣式）。
> - 要改 **介面** → 只能動 `--ui-*`。
> - **`html[data-theme="..."]` 區塊內絕不允許出現 `--doc-*`**。可用下列指令自查（輸出應為空）：
>   ```bash
>   awk '/html\[data-theme=/{f=1} /^  }/{if(f)f=0} f && /--doc-/{print NR": "$0}' seikyusho.html
>   ```

同時把 A4 本體殘留的硬編碼收斂：`var(--gray-bg)`→`--doc-gray-bg`、`var(--header-bg)`→`--doc-header-bg`、`var(--bank-bg)`→`--doc-bank-bg`、印章 `var(--accent)`→`--doc-seal`；`.invoice-page` 的 `color:#111`、明細表格 `border:1px solid #444`、合計上緣 `2px solid #444` 改接 `--doc-ink` / `--doc-line`（值不變，視覺零差異）。

### 4d.3 介面改造

- 55 處硬編碼介面顏色改為 `--ui-*` 引用（以腳本逐條替換並校驗命中次數；`.summary-box .row.total` 原用 `#dde2e6` 而非 `#e2e6ea`，單獨修正）。
- 主題區塊：`:root`（classic）+ `html[data-theme="mono|blue|gray|orange"]`，共 5 版。
- 切換 UI：toolbar 於語言切換左側插入 🎨 下拉選單，5 項各附 `.sw-*` 色票；沿用既有 `.lang-switch` 樣式（`.theme-menu` 加寬至 176px）。
- 行為：`applyTheme(theme, animate)` / `setTheme()` / `toggleThemeMenu()`；與語言選單互斥開啟、點擊外部關閉；切換時掛 `html.theme-switching` 280ms 過渡，`prefers-reduced-motion` 下關閉。
- 無白閃：`</style>` 後插入同步腳本讀 `fig_theme` 並設 `data-theme`。
- **黑金回落**：首頁可能存 `gold`，本頁 `THEMES` 不含該值 → 保持 classic 且**不改寫** `localStorage`（用戶回首頁仍是黑金）。
- i18n：`theme_label / theme_classic / theme_mono / theme_blue / theme_gray / theme_orange`（zh + ja），`applyLang()` 同時刷新觸發按鈕文案。
- **工具列高度同步**：新增 `syncToolbarHeight()`，把 toolbar 實際高度寫入 `--toolbar-h`；`.form-panel` 的 sticky `top` 與 `max-height` 由硬編碼 `61px` 改為 `var(--toolbar-h, 61px)`。原因：加入主題按鈕後 toolbar 更易折行（桌面 62px／行動端 133px 3 行），硬編碼會讓表單面板被工具列遮住。`resize` 與 `document.fonts.ready` 各觸發一次。
- 行動端（≤640px）：`.theme-switch .theme-name { display:none }`，僅留 🎨 + ▾，避免第 4 行。

### 4d.4 對比度修正（WCAG AA）

首輪實測 7 項不達標，全部為「淺灰小字」類令牌：

| 令牌 | 主題 | 原值 → 新值 | 最差背景 | 修正前 → 後 |
|---|---|---|---|---|
| `--ui-muted` | classic | `#6a788c` → `#5c6778` | `--ui-bg` | 3.70 → 4.72 |
| `--ui-muted-2` | classic | `#99a0b0` → `#606977` | `--ui-bg` | 2.16 → 4.57 |
| `--ui-muted-2` | mono | `#757575` → `#6e6e6e` | `--ui-bg` | 4.12 → 4.56 |
| `--ui-muted` | blue | `#5c718c` → `#57698a` | `--ui-bg` | 4.49 → 4.97 |
| `--ui-muted-2` | blue | `#6e8299` → `#5a6d83` | `--ui-bg` | 3.54 → 4.77 |
| `--ui-muted` | gray | `#6f7379` → `#686c73` | `--ui-bg` | 4.34 → 4.80 |
| `--ui-muted-2` | gray | `#7c8087` → `#6a6e75` | `--ui-bg` | 3.61 → 4.66 |
| `--ui-muted-2` | orange | `#8a7259` → `#755d45` | `--ui-bg` | 4.17 → 5.66 |

> 選值時對 **`--ui-bg` / `--ui-card` / `--ui-card-2` / `--ui-sealgen-bg` 四種底色取最差值**，因為同一令牌在這四種底色上都會用到（如 `.hint` 在 bg、`.draft-item .meta span` 在 card、`.modal-body .tip` 在 card2、`.sealgen-hint` 在 sealgen-bg）。

### 4d.5 ⚠️ 驗證陷阱：`Math.random()` 讓跨主題比較完全失效

`seikyusho.html` 每次載入都用 `Math.random()` 重新生成示例資料（第 1079–1080 行 `randInt` / `pick`）。因此**直接比較兩次載入的截圖或 PDF 是無效的** —— 第一次測試得到「5 個主題 5 個不同 A4 哈希」，一度誤判為主題洩漏到紙張。

決定性對照實驗：**同一主題（classic）連跑兩次，差異 318,566 像素**，比跨主題差異（65,738 像素）還大 → 證明差異來源是隨機資料，而非主題。

**正確做法**：在 `page.addInitScript()` 中把 `Math.random` 換成固定種子的 LCG，再比較：

```js
const FREEZE_RANDOM = () => {
  let s = 20261018 >>> 0;
  Math.random = () => { s = (Math.imul(s, 1664525) + 1013904223) >>> 0; return s / 4294967296; };
};
await page.addInitScript(FREEZE_RANDOM);   // 必須在任何頁面腳本之前
```

另一個偽陽性：`locator.screenshot()` 截 `#invoicePage` 時，**sticky 工具列會疊進截圖**（造成 y=12–50 帶狀 ~98,000 像素差異）。截圖前把 `.toolbar` 設 `visibility:hidden`（保留佈局、只去遮擋）即可。

### 4d.6 驗證結果（凍結隨機後）

腳本：`~/.workbuddy-ai/binaries/node/workspace/verify-seikyu-v2.js`（`node verify-seikyu-v2.js 8848`）

| 項目 | 結果 |
|---|---|
| 控制組：同主題兩次載入 | A4 哈希一致、PDF 內嵌圖哈希一致、PDF 位元組數一致（168305）→ **確定性成立** |
| **A4 本體像素哈希（5 主題）** | **唯一值 1 / 5** = `2d19deb37b0effaf` → 主題完全不影響紙張 |
| **PDF 內嵌圖哈希（5 主題）** | **唯一值 1 / 5** = `4a179706`，1588×2246（= 794×1123 @2x） |
| **PDF 頁數 / 尺寸（5 主題）** | **唯一值 1 / 5** = 1 頁 @ `595.3×841.9pt`（= A4） |
| PDF 位元組差異 | 全 5 主題皆 168305 bytes；兩次匯出僅差 **1 個位元組**，位置在 `/CreationDate (D:...)` 時間戳 → 內容完全相同 |
| 介面對比度 | 14 探針 × 5 主題 = **70/70 通過** |
| 工具列高度同步 | 桌面 62px、`--toolbar-h`=62px、`.form-panel` top=62px（5 主題一致） |
| 持久化 | 存 `orange` → 重載 `data-theme="orange"`、按鈕顯示「橘黃」 |
| 黑金回落 | 存 `gold` → `data-theme` 為 null（classic）、按鈕「原版（預設）」、`localStorage` 仍為 `gold` |
| 互動 | 選單 5 項 5 色票、語言選單互斥關閉、點外部關閉、選取後 `active` 唯一 |
| 行動端 390px | toolbar 133px / 3 行 / 無橫向溢出 / 主題名隱藏 |
| 頁面 JS 錯誤 | 無 |

### 4d.7 遺留觀察（非本次改動引入）

- **公司角印文字溢出**：`.seal.company` 為 86×86、`font-size:22px`、`letter-spacing:6px`、`max-height:78px`，示例公司名達 7 字（如「テスト株式会社」）時會超出 78px 被 `overflow:hidden` 裁切。此為既有樣式（本次僅把顏色改接 `--doc-seal`，未動字級／間距），5 主題下表現一致。若要修，建議依字數動態縮放 `font-size`（需重跑 PDF 回歸）。
- **功能欄按鈕數**：加入主題按鈕後桌面仍為單行 62px；≤1280px 起可能折行，已由 `--toolbar-h` 自動補償。

---

## 4e. 變更記錄（2026-10-18）· 明細表頭顏色（5 色）

### 4e.1 需求

在應用頁的設定區（「其他」）新增「明細表頭顏色」切換：**保留原版**，另加 **藍色 / 黑色 / 灰色 / 橘黃**，共 5 色。

與 §4d 的介面主題是**不同維度**，必須分清：

| 維度 | 作用對象 | 存哪 | 影響 PDF |
|---|---|---|---|
| 介面主題（§4d） | 工具列、表單、按鈕 | `localStorage.fig_theme`（跨頁共用） | **不影響** |
| 明細表頭顏色（本節） | 請求書的明細表格表頭 | `state.barTheme`（存進草稿） | **會影響** |

### 4e.2 實作

- **狀態**：`state.barTheme`，預設 `'classic'`；`migrateState()` 對舊草稿（無此欄位）補 `'classic'`。
- **CSS**：以屬性選擇器只覆寫 `--doc-header-bg`，作用域限定在該張請求書：
  ```css
  .invoice-page[data-bar="classic"] { --doc-header-bg: #1e3a5f; }  /* 原版深藍 */
  .invoice-page[data-bar="blue"]    { --doc-header-bg: #1d4ed8; }
  .invoice-page[data-bar="black"]   { --doc-header-bg: #111111; }
  .invoice-page[data-bar="gray"]    { --doc-header-bg: #5b6068; }
  .invoice-page[data-bar="orange"]  { --doc-header-bg: #c2410c; }
  ```
  因為 `--doc-header-bg` 定義在 `:root`，在 `.invoice-page` 上重新賦值即可讓其子孫（`table.items thead th`）取到新值。
- **DOM**：屬性寫在 `#invoicePage` **元素本身**（`setAttribute('data-bar', ...)`），不是 `innerHTML` —— 因此 `renderPreview()` 每次重建內容都不會遺失。`buildExportNode()` 用 `cloneNode(true)`，屬性自動帶到 PDF 用的離線副本。
- **JS**：`BAR_THEMES` / `applyBarTheme()` / `setBarTheme()`；`applyBarTheme()` 在 `renderPreview()` 內呼叫，所以載入草稿、複製上一次、初始化都會自動同步選擇器狀態。
- **UI**：`#barPicker` 5 個 chip（色票 + 文字），沿用 `--ui-*` 令牌，因此它自己也會隨介面主題變色（但不會反向影響表頭）。
- **位置**：獨立成節，標題 `sec_bar`「明細表頭顏色」，放在 **`明細` 區塊正上方**（2026-10-18 調整，原本埋在「其他」區塊底部不易發現）。
- **i18n**：`lbl_bar_color / bar_classic / bar_blue / bar_black / bar_gray / bar_orange / bar_hint`（zh + ja）。

> ⚠️ **不要混淆**：這個功能的 CSS 必須寫 `--doc-header-bg`（紙本），**不可**寫成 `--ui-*`。若誤用 `--ui-*`，PDF 不會變，只會把介面弄花。

### 4e.3 驗證結果

腳本：`~/.workbuddy-ai/binaries/node/workspace/verify-bar.js`（同樣需凍結 `Math.random`）

| 項目 | 結果 |
|---|---|
| 5 個選項實際生效 | `attr` / `--doc-header-bg` / 表頭 `backgroundColor` / 選中狀態 **5/5 全對**，5 色互不相同 |
| **正交性**（固定 bar 色 × 5 個介面主題） | 每種 bar 色的 A4 像素哈希 **唯一值皆為 1**（5×5 = 25 組交叉） |
| **回歸**：`bar=classic` | A4 哈希 = `2d19deb37b0effaf`，與加功能前**完全相同**（預設外觀零變化） |
| PDF | 5 色產生 **5 個不同**內嵌圖哈希；頁數/尺寸仍 **1 頁 @ 595.3×841.9pt** |
| 草稿持久化 | 存 `orange` → 重新載入草稿 → `attr=orange`、表頭 `rgb(194,65,12)`、選中項 `orange` |
| 舊草稿遷移 | 手動刪除 `barTheme` 後 `migrateState()` → 回傳 `classic` |
| i18n | zh「原版/藍色/黑色/灰色/橘黃」、ja「オリジナル/ブルー/ブラック/グレー/オレンジ」；切語言後 bar 選擇保留 |
| 表頭白字對比度 | classic 11.5 / blue 6.70 / black 18.88 / gray 6.33 / orange 5.18 —— **5/5 ≥ AA** |
| 介面回歸 | `verify-seikyu-v2.js` 探針增至 18 項 × 5 主題 = **90/90 通過**；A4 哈希與 PDF 內嵌圖哈希仍唯一 |
| 行動端 | 390px（zh 單行 / ja 2 行）、360px（皆 2 行）；**無橫向溢出**、選擇器自身不溢出 |

### 4e.4 已知取捨

- 日文標籤較長（オリジナル等），在 360–390px 面板寬下無法單行容納 5 個 chip，會折成 2 行（4+1）。已確認無溢出、點擊區 28px 高（與既有 `.mini-btn` 一致），故不強行壓縮字級。
- 表頭色是「文件級」設定但存在**草稿**裡，不是全域偏好。也就是說：不同草稿可以有不同的表頭色（符合「不同客戶／不同案件可用不同樣式」的直覺）；但**不會**跨頁共用。

---

## 4f. 變更記錄（2026-10-18）· 明細表欄寬（単位下拉被裁）＋ 表單欄加寬

### 4f.1 症狀與量化

使用者反映：「明細內容設定裡的單位選擇框太窄，文字看不清楚」。

實測（1440px，表單面板內寬 360px）：

| 欄位 | 欄寬 | 控件實際寬 | 可用文字寬 | 結論 |
|---|---|---|---|---|
| 品目 | 77px | 69px | 55px | ok |
| 数量 | 77px | 69px | 39px | ok |
| **単位** | **37px** | **28.5px** | **-3px** | **「人工」需 26px + 箭頭 18px → 完全放不下** |
| 単価 | 77px | 69px | 39px | ok |

**根因**：`.items-table` 沒有指定欄寬，6 欄交給 `table-layout: auto` 平均分配。`<select>` 的 min-content 只有「最短選項 + 箭頭」，瀏覽器因此把「単位」欄壓到 37px。而 `<select>` 的 `scrollWidth` 不會反映溢出，所以任何「檢查 scrollWidth」的自動化都會漏掉這個問題 —— 必須比對「控件寬 − 內距 − 邊框 − 箭頭」與選項文字實際寬度。

### 4f.2 修法（兩層）

**第一層：明確欄寬 + 下拉硬下限**

```css
.items-table th:nth-child(2) { width: 60px; }   /* 数量：容 3 位數 */
.items-table th:nth-child(3) { width: 68px; }   /* 単位：容 2 個漢字 */
.items-table th:nth-child(4) { width: 72px; }   /* 単価：容 5 位數 */
.items-table select { min-width: 64px; }        /* 硬下限，任何情況都不再被壓扁 */
```

各欄下限的推算公式（寫進註解，日後改字級要重算）：

- 數字框可用文字寬 = 欄寬 − td padding 8 − input padding 12 − 邊框 2 − **數字微調鈕 16**
- 下拉可用文字寬 = 欄寬 − td padding 8 − select padding 12 − 邊框 2 − **下拉箭頭 18**
- 文字框可用文字寬 = 欄寬 − td padding 8 − input padding 12 − 邊框 2

> ⚠️ 那 **16px 的數字微調鈕**很容易被忽略：`input[type=number]` 的上下箭頭會吃掉 16px，若照「文字寬 + 內距」估算，数量／単価 一定會少算。

**第二層：表單欄由 400px → 420px**

360px 內寬根本放不下 6 個可用欄（最小需求約 366px）。實測四種分配後，任何「只調欄寬不加寬」的方案都必然讓某一欄裁切（不是 品目 就是 単価）。因此把 `.layout` 的表單欄上限提高：

```css
.layout { grid-template-columns: minmax(340px, 420px) minmax(0, 1fr); }
.preview-col { overflow-x: auto; }
```

`1fr` → `minmax(0, 1fr)` 是關鍵：舊寫法下 1fr 的 auto 最小値等於預覽內容（794 + 48 = 842），
於是 1101–1261px 區間會把表單欄壓到 340px（品目只剩 14–43px），**且整頁產生橫向溢出**。
改成可收縮後，該區間預覽自動由 `fitInvoice()` 切為行動端回流版面，表單仍保有 420px。

| 視窗寬 | 表單欄 | A4 預覽 | 說明 |
|---|---|---|---|
| ≥1262px | 420px | 完整 A4 794px | 兩欄並排，明細 6 欄全部可讀 |
| 1101–1261px | 420px | 回流（is-mobile） | 修掉了原本的橫向溢出 |
| ≤1100px | 全寬 | 回流 | 單欄堆疊（原行為） |

**第三層：窄螢幕補償**

| 斷點 | 数量 | 単位 | 単価 | 金額 | 削除 | 結果 |
|---|---|---|---|---|---|---|
| 預設（≥1262px） | 60 | 68 | 72 | auto | auto | 品目 84，全欄可讀 |
| ≤640px（手機） | 56 | 66 | 70 | auto | auto | 390px 時品目 78，全欄可讀 |
| ≤380px（SE 等） | 56 | 62 | 68 | 50 | 26 | 360px 時品目 70，全欄可讀 |
| ≤340px（超窄） | 56 | 62 | 68 | **隱藏** | 26 | 320px 時品目 83 |

≤340px 直接隱藏唯讀的「金額」欄（金額仍可由下方小計／合計檢視），避免出現 22px 的無法輸入寬度。

### 4f.3 驗證

腳本：`verify-cols.js`（逐斷點量測欄寬／控件寬／實際文字裁剪）

- 1600 / 1440 / 1280 / 1262px：単位 64px（可用 32px ≥ 需 26px）、品目 76、数量 52、単価 64 —— **全部不裁切**
- 1261 / 1200 / 1101px：表單欄 420、明細全欄可讀、預覽切為回流、**無橫向溢出**
- 390px：単位 58（可用 28 ≥ 24）、品目 70、数量 48、単価 62 —— 全部不裁切
- 375px / 360px：単位 54（可用 24 = 需 24）、品目 77 / 62 —— 全部不裁切
- 320px：隱藏金額欄後品目 83、其餘全可讀
- 回歸：`verify-seikyu-v2.js` **90/90 對比度**、A4 哈希仍為 `2d19deb37b0effaf`（布局改動未觸及紙本）
- 回歸：`verify-bar.js` 5 色生效、25 組正交交叉全通過、草稿持久化正常

---

## 5. 部署与托管

| 项目 | 说明 |
|---|---|
| 源码仓库 | GitHub `takayashoshi888/free-invoice-generator`，分支 `main`，公开 |
| 静态托管 | EdgeOne Pages（Makers），项目 `free-invoice-generator`（ProjectId `makers-yqnficyvptwu`），region `china` |
| 部署方式 | ⚠️ 该项目 Provider 为 **Github**，**只能靠推送 GitHub `main` 触发部署**。`edgeone makers deploy` 的文件夹／zip 上传路径对 GitHub 型项目无效，会报 `Project ... has Provider 'Github'. This project type does not support direct folder or zip file deployment. Only projects with Provider 'Upload' are supported.` 因此**发布流程 = 改文件 → commit → `git push origin main`**，不要尝试用 CLI 直接部署。 |
| 访问监控 | EdgeOne Pages 控制台首页：`访问概览`（24h：流量/请求/带宽峰值/防护次数）、`用量概览`（30 天） |
| 流量下钻 | 点击任一指标可跳转到 EdgeOne 标准「数据分析 › 指标分析」，支持域名/状态码/地区筛选，时间跨度最长 31 天 |

---

## 6. 运维 / 环境注意事项（本机）

- **GitHub 推送**：`git push` 可用系统已存储凭据直接推送，**无需 `gh auth login`**。
- **`gh` CLI（2026-09-30 更新：现已登录）**：账号 `takayashoshi888`（token scopes 含 `repo`），可直接用 `gh api` 做**服务端真值**核验，比只看本地状态可靠：
  ```bash
  gh api repos/takayashoshi888/free-invoice-generator/commits/main --jq '.sha, .commit.message'
  gh api "repos/takayashoshi888/free-invoice-generator/contents/seikyusho.html?ref=main" -q .content | base64 -d
  ```
- **本地落后于远程时**：本地工作副本可能停留在较旧的提交（历史上曾落后 3 个提交）。改动前先 `git fetch origin main && git merge --ff-only FETCH_HEAD`，避免把自己的改动建在旧基底上后覆盖远程更新。
- **EdgeOne 不回写部署状态**：推送后 GitHub 的 `deployments`／`statuses`／`check-runs` 均为空，无法从 GitHub 侧确认 EdgeOne 是否已重新部署；需到 EdgeOne Makers 控制台或线上域名确认。
- **推送验证**：用 GitHub API `contents` 接口解码文件内容核验，而非 `raw.githubusercontent.com`（新推送会短暂 404，非失败）。
- **本地 ref 偶发不落盘**：`git fetch` 后 `git status` 可能显示 `## main...origin/main [gone]`（git 会清理空的 ref 目录）。修复：同一条命令内 `mkdir -p .git/refs/remotes/origin && printf "$(git rev-parse HEAD)\n" > .git/refs/remotes/origin/main`。
- **`/tmp` 不可写**：curl 等写文件请落到工作目录内。
- **`.env`**：当前为空文件，已加入 `.gitignore`，切勿误提交真实密钥。

---

## 7. 后续待办 / 已知问题

- [x] **行动端 PDF 只导出一半**：已修复（commit `20a44fd`，2026-09-30），详见 §4b.1。
- [x] **首頁六版主題切換**：已完成（2026-10-18），详见 §4c.1。
- [x] **應用頁是否套用主題**：已完成（2026-10-18），详见 §4d。採「只換介面」方案，5 版（不含黑金），與首頁共用 `fig_theme`；A4 本體與 PDF 輸出經凍結隨機後驗證為**跨主題完全一致**。
- [x] **明細表頭顏色（5 色）**：已完成（2026-10-18），详见 §4e。文件級設定，存進草稿、會影響 PDF，與介面主題正交（25 組交叉驗證）。
- [x] **單位下拉太窄／明細表欄寬**：已完成（2026-10-18），详见 §4f。下拉 28.5px → 64px，表單欄 400 → 420px，並修掉 1101–1261px 既有的橫向溢出。
- [ ] **窄螢幕明細表**：≤340px 已隱藏「金額」欄換取「品目」可讀；若日後要更好的小螢幕體驗，可考慮把明細改成「每筆一張卡片」的兩行式版面（需重跑 PDF 回歸，因預覽版面會變）。
- [ ] **公司角印長文字溢出**：7 字以上的公司名會被 86×86 印章裁切（既有問題，非主題引入），見 §4d.7。修法建議：依字數動態縮放 `font-size`，改後須重跑 PDF 回歸。
- [ ] **应用页主题的真机复核**：建议在 iPhone Safari 上确认 🎨 菜单在 3 行 toolbar 下的点击热区与遮挡情况（自动化已验证 390px 无溢出，但真机 safe-area 未覆盖）。
- [ ] **确认线上已重新部署**：推送 `20a44fd` 后需到 EdgeOne Makers 控制台确认部署成功，并在真机（iPhone Safari）实测 PDF 导出是否完整。
- [ ] **Cloudflare beacon**：观察 Web Analytics 是否收到数据，必要时改 `defer`。
- [ ] **功能栏按钮数**：`seikyusho.html` 工具栏已 8 个按钮，窄屏（≈1280px 以下）可能折行；已启用 `flex-wrap`，属预期行为非破损。
- [ ] **微信小程序配置**：`project.config.json`（含 appid `wxae8bd4b09efc5ce9`）按用户选择未入库；若改用微信开发者工具托管，需从 `.gitignore` 移除对应两行再提交。

---

## 8. 关键外部链接

| 用途 | 地址 |
|---|---|
| 印章生成器 | https://sealkit.app/kaishain |
| Cloudflare Web Analytics 控制台 | Cloudflare → Web Analytics |
| EdgeOne Pages（Makers）控制台 | Makers 首页 |
| 源码仓库 | https://github.com/takayashoshi888/free-invoice-generator |
