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

## 5. 部署与托管

| 项目 | 说明 |
|---|---|
| 源码仓库 | GitHub `takayashoshi888/free-invoice-generator`，分支 `main`，公开 |
| 静态托管 | EdgeOne Pages（Makers） |
| 访问监控 | EdgeOne Pages 控制台首页：`访问概览`（24h：流量/请求/带宽峰值/防护次数）、`用量概览`（30 天） |
| 流量下钻 | 点击任一指标可跳转到 EdgeOne 标准「数据分析 › 指标分析」，支持域名/状态码/地区筛选，时间跨度最长 31 天 |

---

## 6. 运维 / 环境注意事项（本机）

- **GitHub 推送**：`gh` CLI 未登录，但 `git push` 可用系统已存储凭据直接推送，**无需 `gh auth login`**。
- **推送验证**：用 GitHub API `contents` 接口解码文件内容核验，而非 `raw.githubusercontent.com`（新推送会短暂 404，非失败）。
- **本地 ref 偶发不落盘**：`git fetch` 后 `git status` 可能显示 `## main...origin/main [gone]`（git 会清理空的 ref 目录）。修复：同一条命令内 `mkdir -p .git/refs/remotes/origin && printf "$(git rev-parse HEAD)\n" > .git/refs/remotes/origin/main`。
- **`/tmp` 不可写**：curl 等写文件请落到工作目录内。
- **`.env`**：当前为空文件，已加入 `.gitignore`，切勿误提交真实密钥。

---

## 7. 后续待办 / 已知问题

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
