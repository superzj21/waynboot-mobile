# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目简介

waynboot-mobile 是配套 waynboot-mall 后端的 Vue 3 H5 移动端商城，使用 Vite 6 构建，Vant 4 作为 UI 组件库，Vuex 4 管理状态。Hash 路由 + 部署基础路径 `/mall/`，本地开发访问 `http://localhost:4949/mall/#/`。

详细资料见 [README.md](README.md)，开发规范见 [AGENTS.md](AGENTS.md)，UI 设计规范见 [DESIGN.md](DESIGN.md)。相关项目：后端接口 [waynboot-mall](https://github.com/wayn111/waynboot-mall)、运营后台 [waynboot-admin](https://github.com/wayn111/waynboot-admin)。

## 常用命令

```bash
# 安装依赖
npm install

# 本地开发（端口 4949，/dev-api 与 /upload 代理到 VUE_APP_BASE_URL）
npm run dev

# 生产打包（terser 压缩并 drop_console）
npm run build:prod

# 开发环境打包
npm run build:dev

# Lint（src/ 下的 js 与 vue）
npm run lint

# 运行 Node 回归测试
npm run test:node
# 运行单个测试文件
node --test tests/<文件名>.test.mjs

# 压缩 SVG 图标（增删改 src/icons/svg/ 后执行）
npm run svgo
```

## 核心架构

### 请求与认证

**`src/utils/request.js`** — 全局 Axios 实例。请求拦截器从 cookie 读取 token 注入 `Authorization` header。响应拦截器统一处理：
- `401` → 弹窗提示 + 清除 token + 跳转登录，返回 `Promise.reject`
- `5001` → 静默 reject（订单结果轮询专用）
- `50148` → 直接返回 res（优惠券边界场景）
- 其他非 200 → `showToast` + reject

**`src/utils/auth.js`** — 基于 `js-cookie` 的 token 读写，有效期 15 天。

**`src/permission.js`** — 全局路由守卫。白名单路径（首页、登录、注册、分类、详情、金刚位、搜索、领券中心）无需 token；未登录访问其他路径跳 `/login?redirect=...`。已登录但 Vuex 无用户信息时（页面刷新场景），自动调用 `store.dispatch('user/getInfo')` 补充。`afterEach` 中统一 `closeToast()`。

### 路由

`src/router/index.js` 使用 `createWebHashHistory()` 且全部懒加载。注意：
- 兜底路由 `/:pathMatch(.*)*` 渲染首页 —— 未知路径显示首页，不存在 404 页
- route meta：`title` 由守卫写入 `document.title`；`showTab` 控制 Tabbar 显示；`keepAlive` 配合 [App.vue](src/App.vue) 中的 `<keep-alive>` 缓存页面

### 状态管理

Vuex 4，模块在 `src/store/modules/` 下通过 `import.meta.glob` 自动加载：
- `user` — token（cookie）+ userInfo，提供 `login/logout/getInfo/resetToken` action
- `address` — 地址列表 + 当前选中地址
- `search` — 搜索历史（localStorage）

`src/store/getters.js` 暴露 `token`、`id`、`name`、`userInfo` 等顶层 getter。

### 组件注册

`src/components/` 下四个 setup 函数在 `main.js` 中统一调用：
- `Vant/index.js` — 按需注册所有用到的 Vant 组件，并在 `globalProperties` 上挂载 `$toast`、`$dialog`、`$notify` 封装（内部包装 `showToast`/`showDialog`/`showNotify`）
- `fontAwesome/index.js` — 仅注册 `faSort/faSortUp/faSortDown` 三个图标
- `Wx/index.js` — 挂载 `wxJsSdkInit`，调用后端 `/wx/jsSdkInit` 获取签名并初始化微信 JS SDK
- `common/` — 全局公共组件（image-pic、nav-bar、svg-icon、back-top 等）

新代码中 Vant 弹窗/提示优先直接从 `vant` 导入函数式 API，不用原型挂载的封装。

### 移动端适配

- `src/utils/flexible.js` — 按 750 设计稿将根字号设为 `视口宽度 / 10` px
- `postcss-pxtorem` — 项目样式 `rootValue: 75`，Vant 样式 `37.5`（按文件路径区分）；给元素加 `.ignore` 类可跳过转换。因此 CSS 中写的 px 会被换算，字号需按 H5 实际显示效果校准，不能照搬桌面端尺寸

### 测试

`npm run test:node` 运行 `node --test tests/`，测试的是从页面中抽取的纯逻辑，不是 UI 测试。约定：把可测的纯函数抽到视图目录下的 `.mjs` 文件（如 `src/views/seckill/seckill-payment-flow.mjs`、`src/views/detail/modules/skuState.mjs`），测试文件放 `tests/*.test.mjs` 直接 import。改造页面逻辑时保持这个抽取模式。

### 工具函数

- `src/utils/index.js` — `toDecimal2`（保留两位小数）、`Valid.checkPhone`（手机号校验）、`pathMatcher`（白名单前缀匹配，`**` 通配）
- `src/filter/index.js` — `yuan`（金额格式化）、`dateFormat`（时间戳转日期），挂载到 `globalProperties`

### 打包配置

`vite.config.js` 中 `manualChunks` 将 `vue/vue-router/vuex`、`vant`、`weixin-js-sdk` 拆成独立 chunk；生产构建启用 terser 并 `drop_console`。开发服务器将 `/dev-api`、`/upload` 代理到 `VUE_APP_BASE_URL`（默认 `http://localhost:82`）。

### SVG 图标

使用自定义 `createSvgIconsPlugin`（`build/svg-icons-plugin.js`），图标放在 `src/icons/svg/`，通过 `<svg-icon name="xxx">` 组件使用，入口通过 `virtual:svg-icons-register` 注册。

### 环境变量

使用 `VUE_APP_` 前缀（非标准 Vite 的 `VITE_` 前缀），在 `vite.config.js` 中通过 `envPrefix` 配置同时支持两者。后端接口地址通过 `VUE_APP_BASE_API` 配置，开发时通过 `/dev-api` 代理转发。

## 设计约束（重要）

商城前台 UI 以 [DESIGN.md](DESIGN.md) 和 [AGENTS.md](AGENTS.md) 为准，落地为浅色 Apple 风格 H5，改动 UI 前必须先读这两个文件。核心约束：

- 全站浅色：页面背景 `#f5f5f7`/白色/浅灰，主文字 `#1d1d1f`，交互主色 `#0071e3`；**禁止**商城前台页面大面积黑底
- 设计 token 统一使用 `src/styles/design-system.scss` 中的 `--wb-*` CSS 变量（`--wb-surface`、`--wb-text`、`--wb-blue`、`--wb-radius-card` 等）
- 卡片遵循"单层容器"原则，禁止玻璃嵌套玻璃、卡片里套卡片；阴影只用轻量单层
- 常规文字字号不低于约 28px（pxtorem 下按 H5 实际显示校准）；固定底栏、Tabbar、结算栏不得遮挡内容
- 禁止与商城功能无关的品牌口号、装饰性文案和占位描述
- UI 改动后必须用移动端 viewport、移动端 UA 和 touch 环境自检，提交前保留截图

## 开发约定

- 新页面使用 Vue 3 `<script setup>`，不用 `this`；通过 `useRouter`/`useRoute`/`useStore` 获取实例
- 代码风格由 eslint 强制：单引号、无分号、`===`、2 空格缩进
- 页面放 `src/views/<module>/`，页面私有的子组件放该页面的 `modules/` 目录；跨页面复用的放 `src/components/`；接口按业务域放 `src/api/`
- Commit message 用中文祈使句带作用域前缀，如 `feat(订单): 新增确认订单页面`、`fix(购物车): 修复全选金额错误`；一次提交只解决一个主要问题
- 提交前运行 `npm run lint` 和 `npm run test:node`（涉及打包配置的改动另跑 `npm run build:prod`）
