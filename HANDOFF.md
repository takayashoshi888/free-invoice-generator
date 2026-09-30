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
