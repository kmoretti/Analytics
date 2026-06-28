# Han Analytics — Code Wiki

> Han Analytics 是一个轻量级、零成本的网站分析平台，托管在 Cloudflare Pages 上，利用 Cloudflare Analytics Engine 作为数据存储，无需自备服务器或数据库。

---

## 目录

- [项目概述](#项目概述)
- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [整体架构](#整体架构)
- [模块详解](#模块详解)
  - [1. 客户端追踪脚本 (tracker.js)](#1-客户端追踪脚本-trackerjs)
  - [2. 数据上报端点 (functions/send.js)](#2-数据上报端点-functionssendjs)
  - [3. 数据查询 API (functions/api.js)](#3-数据查询-api-functionsapijs)
  - [4. 数据处理工具 (functions/utils/)](#4-数据处理工具-functionsutils)
  - [5. 前端仪表板 (src/App.vue)](#5-前端仪表板-srcappvue)
  - [6. UI 组件库 (src/components/ui/)](#6-ui-组件库-srccomponentsui)
- [环境变量与绑定](#环境变量与绑定)
- [依赖说明](#依赖说明)
- [工作流程](#工作流程)
- [运行方式](#运行方式)

---

## 项目概述

Han Analytics 是一个**简单优雅的 Web 分析仪表板**。它通过在目标网站嵌入一段轻量级 JavaScript 脚本，自动采集页面访问数据，然后将数据发送到 Cloudflare Pages Functions 写入 Cloudflare Analytics Engine，最终在仪表板中以图表和列表形式展示访问量、访客数、页面排行、来源渠道、浏览器、操作系统、地理分布等统计数据。

| 特性 | 说明 |
|------|------|
| 数据存储 | Cloudflare Analytics Engine（无服务器、无需数据库） |
| 成本 | 零成本运行，每天支持约 10 万次免费统计 |
| 部署 | 一键部署到 Cloudflare Pages |
| 安全 | 支持密码访问和网站白名单 |

## 技术栈

| 层级 | 技术 | 用途 |
|------|------|------|
| **前端框架** | Vue 3 (Composition API + `<script setup>`) | 构建单页应用仪表板 |
| **构建工具** | Vite 6 | 构建与开发服务器 |
| **语言** | TypeScript / JavaScript | 前后端代码 |
| **样式** | Tailwind CSS 3 + Less | UI 样式 |
| **UI 组件** | shadcn-vue (基于 Radix Vue) | 开箱即用的 UI 组件 |
| **图表** | ECharts 5 (SVG 渲染) | 趋势折线图 |
| **后端** | Cloudflare Pages Functions | 无服务器 API 端点 |
| **数据存储** | Cloudflare Analytics Engine SQL API | 数据写入与查询 |
| **包管理** | pnpm | 依赖管理 |

---

## 项目结构

```
Analytics/
├── functions/                    # Cloudflare Pages Functions（服务端）
│   ├── api.js                    # 数据查询 API 入口
│   ├── send.js                   # 数据上报 API 入口
│   └── utils/                    # 工具函数
│       ├── area.js               # 国家/地区代码 → 中文名称映射表
│       ├── index.js              # 时间格式化、数据聚合、图表数据处理
│       └── init.js               # 核心查询逻辑，组装 SQL 调用 CF Analytics Engine
├── public/                       # 静态资源（直接部署）
│   ├── icon/                     # 国家/地区/操作系统图标 PNG
│   ├── tracker.js                # 客户端追踪脚本（完整版）
│   ├── tracker.min.js            # 客户端追踪脚本（压缩版）
│   └── favicon.ico
├── src/                          # Vue 前端源码
│   ├── assets/
│   │   ├── font/                 # 字体文件 (woff2)
│   │   ├── svg/                  # SVG 图标（底部技术栈展示）
│   │   ├── font.less             # 字体定义
│   │   ├── index.less            # 页面主样式
│   │   └── main.css              # Tailwind 导入 + CSS 变量 + 全局样式
│   ├── components/ui/            # shadcn-vue UI 组件
│   │   ├── alert/                # Alert 组件
│   │   ├── alert-dialog/         # AlertDialog 模态框
│   │   ├── button/               # Button 按钮
│   │   ├── card/                 # Card 卡片
│   │   ├── input/                # Input 输入框
│   │   ├── scroll-area/          # ScrollArea 滚动区域
│   │   ├── select/               # Select 下拉选择器
│   │   ├── skeleton/             # Skeleton 骨架屏
│   │   └── toast/                # Toast 消息提示
│   ├── lib/
│   │   └── utils.ts              # cn() 工具函数（Tailwind class 合并）
│   ├── App.vue                   # 主应用组件（单页，所有逻辑在此）
│   └── main.ts                   # Vue 应用入口
├── components.json               # shadcn-vue 配置
├── tailwind.config.js            # Tailwind CSS 配置
├── vite.config.ts                # Vite 构建配置
├── tsconfig*.json                # TypeScript 配置
├── eslintrc.cjs                  # ESLint 配置
├── .prettierrc.json              # Prettier 配置
└── package.json                  # 项目依赖与脚本
```

---

## 整体架构

Han Analytics 采用**前后端分离 + 无服务器架构**，完整数据流如下：

```
[用户浏览器]                  [目标网站]                [Cloudflare Pages]
    │                            │                           │
    │  1. 嵌入 tracker.js        │                           │
    │◄──── 加载追踪脚本 ────────┤                           │
    │                            │                           │
    │  2. 页面访问时自动上报      │                           │
    ├──────── POST /send ────────┼──── Cloudflare Function ► │
    │                            │                           │
    │                            │      3. 写入数据           │
    │                            │        ▼                  │
    │                            │  Cloudflare Analytics     │
    │                            │  Engine (AnalyticsBinding)│
    │                            │                           │
    │  4. 管理员访问仪表板        │                           │
    ├──────── POST /api ─────────┼──── Cloudflare Function ► │
    │                            │      5. 查询数据           │
    │                            │        ▼                  │
    │                            │  Cloudflare Analytics     │
    │  6. 渲染图表与列表 ◄───────┼── Engine SQL API ◄─────────│
    │                            │                           │
```

### 数据流关键路径

1. **数据采集**：目标网站加载 `tracker.js` → 脚本自动分析访客信息 → 发送 POST 请求到 `/send`
2. **数据存储**：`functions/send.js` 通过 `AnalyticsBinding.writeDataPoint()` 将数据写入 Cloudflare Analytics Engine
3. **数据查询**：仪表板发起 POST 请求到 `/api` → `functions/api.js` 调用 CF Analytics Engine SQL API 查询 → 返回处理后的 JSON
4. **数据可视化**：前端接收 JSON → ECharts 渲染趋势折线图 / ScrollArea 渲染排行列表

---

## 模块详解

### 1. 客户端追踪脚本 (tracker.js)

**路径**: `public/tracker.js`（完整版）、`public/tracker.min.js`（压缩版）

**功能**: 嵌入目标网站后，自动采集并上报页面访问数据。

**核心逻辑**:

| 功能 | 实现方式 |
|------|----------|
| **网站标识** | 通过 `<script>` 标签的 `data-website-id` 属性获取 |
| **页面路径** | `window.location.pathname`，URL 编码后上报 |
| **来源地址** | `document.referrer`，与当前域名相同则视为站内跳转，不上报 |
| **访客去重** | 用 `localStorage` 记录最后访问日期，同一天内不重复计为"新访客" |
| **访问间隔** | 用 `localStorage` 记录每个页面的最后访问时间，默认 30 分钟内不重复计为"新访问" |
| **上报方式** | `fetch POST` 异步上报到 `/send` 端点，静默失败（无 `catch` 处理） |
| **Electron 检测** | 检查 `navigator.userAgent` 包含 `Electron` 则跳过 |

**关键变量**:
- `visitor` (Boolean): 是否为今日首次访问（去重新访客）
- `visit` (Boolean): 是否在 30 分钟访问间隔之外（去重新访问）
- `AccessInterval` (Number): 访问间隔阈值，默认 30 分钟
- `PathVisit` (Object): 存储各页面最后访问时间戳，持久化到 `localStorage._vhLastVisit`

---

### 2. 数据上报端点 (functions/send.js)

**路径**: `functions/send.js`

**功能**: 接收追踪脚本上报的数据，写入 Cloudflare Analytics Engine。

**请求方式**: `POST`

**请求体结构**:
```json
{
  "website": "网站唯一标识",
  "host": "域名",
  "path": "页面路径",
  "referrer": "来源地址",
  "visitor": true/false,
  "visit": true/false
}
```

**关键逻辑**:

| 步骤 | 说明 |
|------|------|
| **白名单校验** | 如果设置了 `CLOUDFLARE_WEBSITE_WHITELIST`，校验 website + host 是否在白名单中 |
| **UA 解析** | 使用 `ua-parser-js` 库解析 User-Agent，提取浏览器名称和操作系统名称 |
| **地理定位** | 通过 `request.cf.country` 获取访客国家/地区代码 |
| **来源处理** | 如果 referrer 域名与当前 host 相同，视为站内跳转，设为空字符串 |
| **数据写入** | 调用 `env.AnalyticsBinding.writeDataPoint()` 写入 Cloudflare Analytics Engine |

**DataPoint 写入结构**:

| 字段 | 数据类型 | 内容 | 说明 |
|------|----------|------|------|
| blob1 | blob | website | 网站唯一标识 |
| blob2 | blob | host | 域名 |
| blob3 | blob | path | 页面路径 |
| blob4 | blob | referrerUrl | 来源地址 |
| blob5 | blob | osName | 操作系统名称 |
| blob6 | blob | browserName | 浏览器名称 |
| blob7 | blob | areaCode | 国家/地区代码 |
| blob8 | blob | userAgent | 原始 UA 字符串 |
| double1 | double | visitor | 1 或 0（新访客标志） |
| double2 | double | visit | 1 或 0（新访问标志） |

---

### 3. 数据查询 API (functions/api.js)

**路径**: `functions/api.js`

**功能**: 提供仪表板所需的各种数据查询接口。

**请求方式**: `POST`

**请求体结构**:
```json
{
  "type": "visit|list|path|referrer|os|soft|area|echarts",
  "siteID": "网站标识",
  "time": "today|1d|7d|30d|60d|90d",
  "session": "登录密码（可选）"
}
```

**支持的查询类型 (type)**:

| type | 返回值 | 说明 |
|------|--------|------|
| `visit` | `{ views, visitor, visit }` | 总浏览量、独立访客数、访问次数 |
| `list` | `string[]` | 所有已注册的网站标识列表 |
| `path` | `{ name, value, per }[]` | 页面访问排行 |
| `referrer` | `{ name, value, per }[]` | 来源渠道排行 |
| `os` | `{ name, value, per }[]` | 操作系统排行 |
| `soft` | `{ name, value, per }[]` | 浏览器排行 |
| `area` | `{ name, value, per, code }[]` | 地区分布排行 |
| `echarts` | `{ name, value }[]` | 趋势图时序数据 |

**安全机制**:
- 如果设置 `CLOUDFLARE_WEBSITE_PWD`，请求必须携带正确的 `session` 密码
- `type=Login` 用于单独的登录验证

**时间周期 (time)**:
- `today`: 当天从 00:00 到当前小时
- `1d`: 过去 24 小时
- `7d` / `30d` / `60d` / `90d`: 过去 N 天

---

### 4. 数据处理工具 (functions/utils/)

#### 4.1 init.js — 核心查询逻辑

**路径**: `functions/utils/init.js`

**导出函数**: `vh_INIT(env, time, siteID, tz, type)`

**功能**: 根据查询类型，组装不同的 SQL 查询语句，通过 Cloudflare Analytics Engine SQL API 获取数据，并进行后处理。

**SQL 查询示例**:

```sql
-- visit: 聚合统计
SELECT SUM(_sample_interval) AS views,
       SUM(IF(double1 = '1', double1, 0.0)) AS visitor,
       SUM(IF(double2 = '1', double2, 0.0)) AS visit
FROM AnalyticsDataset
WHERE timestamp >= ... AND blob1 = 'siteID'

-- echarts: 按小时/天分组
SELECT formatDateTime(timestamp, '%Y-%m-%d %H:00:00') AS hour,
       SUM(_sample_interval) AS count
FROM AnalyticsDataset
WHERE ... GROUP BY hour ORDER BY hour

-- path/referrer/os/soft/area: 提取维度值
SELECT blob3 FROM AnalyticsDataset WHERE ...
```

**API 端点**: `https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/analytics_engine/sql`

**认证方式**: `Authorization: Bearer {CLOUDFLARE_API_TOKEN}`

#### 4.2 index.js — 数据处理函数

**路径**: `functions/utils/index.js`

| 函数 | 参数 | 功能 |
|------|------|------|
| `formatTime(timeStr, tz)` | 时间周期字符串、时区 | 将 `today/1d/7d/...` 格式转换为 SQL 时间范围条件 |
| `countData(arr, key, keyType, status)` | 原始数据数组、维度键名、时间配置、是否排序 | 对查询结果按维度聚合计数，计算百分比，生成带 `name/value/per` 的排行榜数据 |
| `echartsData(data, key, tz)` | 原始数据数组、时间周期、时区 | 处理趋势图数据，按小时/日分组，填充缺失时间点 |

**`countData` 输出格式**:
```json
[
  { "name": "/index.html", "value": "1.2K", "per": "25%" },
  { "name": "/about", "value": "800", "per": "16%" }
]
```

当 value >= 1000 时自动转换为 `X.XK` 格式。

#### 4.3 area.js — 地区映射表

**路径**: `functions/utils/area.js`

**导出常量**: `AREAS`

**功能**: 包含 ISO 3166-1 两位国家/地区代码到中文名称的映射。例如：`CN → 中国`、`US → 美国`、`JP → 日本`。用于将 `request.cf.country` 获取的代码转换为可读的中文名称。

---

### 5. 前端仪表板 (src/App.vue)

**路径**: `src/App.vue`

**文件大小**: ~460 行（单文件组件，含模板、脚本、样式）

**功能**: 整个仪表板的所有 UI 逻辑，包括站点选择、周期选择、数据加载、图表渲染、各维度排行展示、登录鉴权。

**主要状态变量**:

| 变量 | 类型 | 用途 |
|------|------|------|
| `authStatus` | ref\<boolean> | 控制登录弹窗是否显示 |
| `session` | ref\<string> | 从 localStorage 读取的登录密码 |
| `siteList` | ref\<string[]> | 所有已注册的网站标识列表 |
| `siteValue` | ref\<string> | 当前选中的网站标识 |
| `timeValue` | ref\<string> | 当前选中的时间周期 |
| `resData` | ref\<object> | 查询到的全部统计数据 |
| `getDatasStatus` | ref\<boolean> | 数据加载状态，用于禁用交互 |

**核心方法**:

| 方法 | 功能 |
|------|------|
| `loginFn()` | 密码登录验证，调用 `/api` type=Login，成功后保存 session 到 localStorage |
| `getSiteList()` | 获取网站列表，调用 `/api` type=list；若返回 401 则弹出登录框 |
| `getDatas()` | 并发请求 7 种类型的数据（visit/path/referrer/os/soft/area/echarts），使用 `Promise.all` 减少等待时间 |
| `renderEcharts(dateList, valueList)` | 使用 ECharts 渲染趋势折线图，配置 SVG 渲染器、平滑曲线、渐变填充 |
| `getIconUrl(url)` | 通过 DuckDuckGo 图标服务获取来源网站 favicon |
| `getIcon(code)` | 加载本地 `/icon/{code}.png` 显示地区/操作系统图标 |

**页面布局**:

```
┌────────────────────────────────────────────────────┐
│  Header: Han Analytics 简单优雅的Web分析            │
├────────────────────────────────────────────────────┤
│  Alert: 项目介绍 + GitHub 链接                      │
├────────────────────────────────────────────────────┤
│  站点选择 ▼  周期选择 ▼    |  Views  Visitors Visits│
├────────────────────────────────────────────────────┤
│              [趋势折线图 (ECharts)]                  │
├─────────────────────┬──────────────────────────────┤
│    Pages 排行        │    Referrers 来源排行         │
├─────────────────────┼──────────────────────────────┤
│    Browsers 排行     │    OS 排行    │  Areas 地区排行│
├─────────────────────┴──────────────────────────────┤
│  Footer: Cloudflare / Vue / 技术栈展示              │
└────────────────────────────────────────────────────┘
```

**数据加载策略**: 使用 `Promise.all` 并行请求 7 个接口，每个接口完成后更新 `tempResData`，待所有请求完成后再一次性赋值给 `resData` 触发渲染更新，避免频繁 DOM 更新。

---

### 6. UI 组件库 (src/components/ui/)

基于 **shadcn-vue** 生成的可复用 UI 组件，底层依赖 **Radix Vue**（无头 UI 组件库）和 **Tailwind CSS**。

| 组件 | 功能 |
|------|------|
| **Alert** | 信息提示条，用于显示项目介绍 |
| **AlertDialog** | 模态弹窗，用于密码登录 |
| **Button** | 按钮组件 |
| **Card** | 卡片容器，用于各排行列表 |
| **Input** | 文本输入框，用于登录密码输入 |
| **ScrollArea** | 自定义滚动区域，用于排行列表滚动 |
| **Select** | 下拉选择器，用于站点和周期选择 |
| **Skeleton** | 骨架屏，数据加载时的占位符 |
| **Toast** | 消息提示，用于操作反馈 |

---

## 环境变量与绑定

### 环境变量 (Cloudflare Pages 设置)

| 变量名 | 必需 | 说明 |
|--------|------|------|
| `CLOUDFLARE_ACCOUNT_ID` | 是 | Cloudflare 账户 ID（可在 Workers 页面找到） |
| `CLOUDFLARE_API_TOKEN` | 是 | Cloudflare API 令牌（需要 Analytics Engine 读写权限） |
| `CLOUDFLARE_WEBSITE_PWD` | 否 | 仪表板访问密码（不设置则公开访问） |
| `CLOUDFLARE_WEBSITE_WHITELIST` | 否 | 可统计的网站白名单，格式：`域名,WebsiteID\|域名,WebsiteID` |

### 绑定 (Cloudflare Pages 绑定)

| 绑定类型 | 变量名 | 数据集 |
|----------|--------|--------|
| Analytics Engine | `AnalyticsBinding` | `AnalyticsDataset` |

---

## 依赖说明

### 生产依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `vue` | ^3.5.13 | 前端框架 |
| `echarts` | ^5.6.0 | 趋势图渲染 |
| `radix-vue` | ^1.9.15 | 无头 UI 组件库 |
| `lucide-vue-next` | ^0.475.0 | 图标库 |
| `@radix-icons/vue` | ^1.0.0 | Radix 图标 |
| `@vueuse/core` | ^12.7.0 | Vue 组合式工具 |
| `class-variance-authority` | ^0.7.1 | CSS 变体管理 |
| `clsx` | ^2.1.1 | className 拼接 |
| `tailwind-merge` | ^2.6.0 | Tailwind class 合并 |
| `tailwindcss-animate` | ^1.0.7 | Tailwind 动画插件 |
| `dayjs` | ^1.11.13 | 时间处理（服务端） |
| `ua-parser-js` | ^2.0.2 | User-Agent 解析（服务端） |
| `vh-plugin` | ^1.2.2 | Vue 插件工具（加载动画等） |

### 开发依赖

| 包名 | 用途 |
|------|------|
| `vite` ^6.1.0 | 构建工具 |
| `vue-tsc` | Vue TypeScript 类型检查 |
| `typescript` ~5.7.3 | TypeScript 编译器 |
| `tailwindcss` ^3.4.17 | CSS 框架 |
| `less` ^4.2.2 | Less 预处理器 |
| `autoprefixer` | CSS 自动添加浏览器前缀 |
| `eslint` | 代码检查 |
| `prettier` | 代码格式化 |
| `npm-run-all2` | 并行运行 npm 脚本 |
| `@vitejs/plugin-vue` | Vite Vue 插件 |

---

## 工作流程

### 数据采集流程

```
用户访问目标网站
        │
        ▼
加载 tracker.js 脚本
        │
        ├─ 读取 data-website-id 获取网站标识
        ├─ 从 localStorage 判断访客/访问去重
        ├─ 提取 host、pathname、referrer
        │
        ▼
发送 POST /send 请求
        │
        ▼
functions/send.js 处理
        ├─ 白名单校验（可选）
        ├─ UA 解析（浏览器 + 操作系统）
        ├─ 获取国家代码
        ├─ 处理来源地址
        │
        ▼
AnalyticsBinding.writeDataPoint()
        │
        ▼
Cloudflare Analytics Engine
```

### 数据展示流程

```
管理员访问仪表板页面
        │
        ▼
App.vue onMounted()
        ├─ 初始化 ECharts 实例
        ├─ 调用 getSiteList()
        │
        ▼
发送 POST /api (type=list)
        │
        ▼
返回网站列表 → 默认选中第一个
        │
        ▼
调用 getDatas()
        ├─ 并行发送 7 个 POST /api 请求
        │   (visit/path/referrer/os/soft/area/echarts)
        │
        ▼
functions/api.js → vh_INIT()
        ├─ 组装 SQL 查询
        ├─ 调用 CF Analytics Engine SQL API
        ├─ 数据聚合处理 (countData / echartsData)
        │
        ▼
返回 JSON → 渲染 Dashboard
        ├─ ECharts 趋势折线图
        ├─ Pages / Referrers / Browsers / OS / Areas 排行
        └─ Views / Visitors / Visits 汇总数字
```

---

## 运行方式

### 本地开发

```bash
# 1. 安装依赖
pnpm install

# 2. 启动开发服务器（前端热更新）
pnpm dev
# 默认地址: http://localhost:52101

# 注意: 本地开发时 Functions 和 Analytics Engine 需通过 wrangler 模拟
pnpm preview  # 构建后通过 wrangler pages dev 来模拟生产环境
```

### 构建

```bash
pnpm build
# 构建产物输出到 dist/ 目录
# 会同时执行 type-check 和 build-only
```

### 部署到 Cloudflare Pages

1. Fork 此仓库或使用模板创建新仓库
2. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
3. 创建 Pages 项目，关联 GitHub 仓库
4. 构建配置：框架选择 **Vue**，构建命令 `pnpm build`，输出目录 `dist`
5. 在环境变量中设置：
   - `CLOUDFLARE_ACCOUNT_ID`
   - `CLOUDFLARE_API_TOKEN`
   - （可选）`CLOUDFLARE_WEBSITE_PWD`
   - （可选）`CLOUDFLARE_WEBSITE_WHITELIST`
6. 在项目设置 → 绑定中，添加 **Analytics Engine** 绑定：
   - 变量名称: `AnalyticsBinding`
   - 数据集: `AnalyticsDataset`
7. 重新部署后访问 `https://{your-project}.pages.dev`

### 集成到目标网站

在需要统计的网站页面底部添加：

```html
<script defer src="https://your-project.pages.dev/tracker.min.js" data-website-id="your-site-id"></script>
```

将 `data-website-id` 设置为该网站的唯一标识字符串（如 `my-blog`），仪表板中将以此标识区分不同站点的数据。

### NPM Scripts

| 命令 | 功能 |
|------|------|
| `pnpm dev` | 启动 Vite 开发服务器，端口 52101 |
| `pnpm build` | 类型检查 + 构建生产版本 |
| `pnpm build-only` | 仅构建不检查类型 |
| `pnpm type-check` | Vue TypeScript 类型检查 |
| `pnpm preview` | 构建后用 wrangler 模拟 Pages 环境（含 Functions） |
| `pnpm lint` | ESLint 代码检查 |
| `pnpm format` | Prettier 代码格式化 |

---

> 本文档基于 commit 时的代码版本自动生成，如有更新请同步更新。
