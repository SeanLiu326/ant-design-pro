# Ant Design Pro 开发上手指南

面向刚加入项目 / 刚 clone 下代码的同学。目标是：**半小时内本地跑起来，半天内提交第一个 PR**。

- 仓库约定（AI 与人都要遵守）：[`CLAUDE.md`](../CLAUDE.md)
- 深度速查（API、配置、示例代码大全）：[`docs/cheatsheet.zh-CN.md`](./cheatsheet.zh-CN.md)
- 本指南只讲「怎么上手 + 别踩什么坑」，API 细节请查上面两份。

---

## 0. 前置要求

| 项 | 要求 | 说明 |
| --- | --- | --- |
| Node | **≥ 22**（`package.json` 的 `engines`） | 本地实测 Node 24 可用 |
| 包管理器 | **npm** | 仓库只维护 `package-lock.json`。CI 用 `npm ci` |
| 编辑器 | VS Code / WebStorm 均可 | 建议装 Biome 插件；`.editorconfig` 已规定 2 空格 + LF |

`.npmrc` 里设了 `legacy-peer-deps=true`，所以 `npm install` 直接跑，不用加参数。

---

## 1. 三分钟跑起来

```bash
git clone --depth=1 https://github.com/ant-design/ant-design-pro.git my-app
cd my-app
npm install        # 会顺带执行 prepare：安装 husky 钩子 + umi setup 生成类型
npm start          # UMI_ENV=dev，带 Mock，默认 http://localhost:8000
```

打开 http://localhost:8000 ，用下面任一账号登录：

| 账号 | 密码 | 权限 |
| --- | --- | --- |
| `admin` | `ant.design` | `access = admin` → `canAdmin` 为真，能看到「管理页」菜单 |
| `user` | `ant.design` | 普通用户，管理页菜单不出现 |

> 账号逻辑写在 `mock/user.ts`，权限判断在 `src/access.ts`。

两个值得知道的命令：

- `npm run build && npm run preview` —— 用构建产物在 8000 端口本地预览（排查「只有构建后才出现的问题」，比如按需拆分、静态资源路径）。
- `npm run analyze` —— 产物体积分析，加依赖前先看看。

---

## 2. 目录地图：哪些能改，哪些不能

```
config/
  config.ts          # 主配置：插件、主题、代理、mock 范围、构建（utoopack）
  routes.ts          # 声明式路由 = 菜单来源，加页面先改这里
  proxy.ts           # 按 UMI_ENV 分环境的后端代理
  defaultSettings.ts # ProLayout 外观（主题色、布局模式、Logo）
  oneapi.json        # OpenAPI schema，npm run openapi 用它生成 services
mock/                # 全局 mock：user/listTableList/notices/route/utils
src/
  app.tsx            # 运行时配置：getInitialState、layout、request、rootContainer
  access.ts          # 权限定义（canAdmin）
  requestErrorConfig.ts # 统一错误处理：业务错误码 / HTTP 错误 / 断网
  global.tsx         # 仅引入 tailwind.css 的副作用入口
  global.less        # 全局字体、少量全局样式（历史遗留）
  global.style.ts    # 全局 antd-style 样式
  loading.tsx        # 路由懒加载的骨架屏
  typings.d.ts       # 静态资源与全局常量（__APP_VERSION__ 等）声明
  components/        # 公共组件，统一从 components/index.ts 导出
  pages/             # 页面（内聚目录，见 §5）
  services/ant-design-pro/  # ⚠ 自动生成，禁止手改
  locales/           # 8 种语言：zh-CN en-US zh-TW ja-JP id-ID bn-BD pt-BR fa-IR
  utils/             # 通用工具（chinaDivision、format）
types/index.d.ts     # 全局 API.* 命名空间（接口出入参类型）
cloudflare-worker/   # 演示后端（Hono + Cloudflare Workers），独立子项目
docs/                # 文档与 roadmap
```

### 三条「不要碰」

1. **`src/services/ant-design-pro/**` 是自动生成的**，文件头有 `// @ts-ignore` + `/* eslint-disable */`。要改接口就改 `config/oneapi.json` 后 `npm run openapi` 重新生成。
2. **`src/.umi/**` / `.umi-test/**` 是生成目录**，别手改也不进 git。dev server 行为诡异时，删掉 `src/.umi` 重启。
3. **不要在 `mock/` 和 `src/services/` 里纠结代码风格** —— Biome 配置（`biome.json`）已经忽略它们。

---

## 3. 命令速查（都是 `package.json` 里真实存在的）

**开发**

| 命令 | 含义 |
| --- | --- |
| `npm start` | `UMI_ENV=dev`，**带 Mock**，日常开发用这个 |
| `npm run dev` | `UMI_ENV=dev` + `MOCK=none`，请求真实接口 |
| `npm run start:test` | `UMI_ENV=test` + 无 Mock，`/api` 代理到 `https://pro-api.ant-design-demo.workers.dev` |
| `npm run start:pre` | `UMI_ENV=pre`，代理地址在 `config/proxy.ts` 的 `pre` 里（当前是占位符，需自己填） |
| `npm run start:no-mock` | 等价于 `npm run dev` |

**质量**

| 命令 | 含义 |
| --- | --- |
| `npm run lint` | `biome lint` + `tsc --noEmit`，提交前必须过 |
| `npm run biome` | `biome check --write`，自动修复格式与可修问题 |
| `npm test` | Vitest 单次运行（**不是 Jest**） |
| `npm run test:watch` / `test:ui` / `test:coverage` | 监听 / 可视化 / 覆盖率 |
| `npx vitest run -u` | 更新快照（没有 `npm run test:update` 这个脚本） |
| `npm run doctor` | React Doctor 静态体检，CI 里会跑 |
| `npx antd info <组件>` | 查 antd 组件 API，写 antd 代码前**必做** |
| `npx antd lint ./src` | antd 用法检查（废弃 API / a11y / usage），提交前必须过 |

**构建与生成**

| 命令 | 含义 |
| --- | --- |
| `npm run build` | 生产构建（utoopack），产物在 `dist/` |
| `npm run preview:build` | 构建 + 本地预览（8000 端口） |
| `npm run analyze` | 构建 + 体积分析 |
| `npm run openapi` | 按 schema 重新生成 `src/services/` |
| `npm run i18n-remove` | 移除国际化（写死 zh-CN），慎用 |
| `npm run simple` | ⚠ 切换为精简模板，**不可逆**，先提交代码 |
| `npm run record` | 录制登录场景请求，供 request-record 回放 |

---

## 4. 六条硬约定

1. **只认 Biome**：没有 ESLint，没有 Prettier。手写代码时单引号、2 空格（`biome.json` 已配置）。
2. **写 antd 代码前先 `npx antd info <Component>`**，不要凭记忆写 props；v6 有大量 API 变更（例如 `List` 已废弃，官方推荐 `Listy`）。
3. **提交信息用 Conventional Commits**：`feat: xxx` / `fix: xxx` / `chore(deps): ...`。husky 的 `commit-msg` 钩子会用 commitlint 拦截不合规信息。
4. **pre-commit 会改写你的代码**：husky 的 `pre-commit` 对暂存的 `js/ts/tsx/css/less`（以及 `src/**/*.md`）跑 `biome check --write`，被改写后需要重新 `git add`。
5. **TS strict**：`noImplicitReturns` 也开着；类型写在 `types/`（全局 `API.*`）或页面内 `data.d.ts`。
6. **Node ≥ 22 + npm**：不要 `pnpm add` / `yarn`。

---

## 5. 实战：端到端加一个页面

以新增「报表页 `/report`」为例，共 4 步。

### 第 1 步：建页面目录（页面内聚）

一个页面目录长这样（参照 `src/pages/table-list/`、`src/pages/dashboard/analysis/`）：

```
src/pages/report/
  index.tsx        # 页面组件，默认导出
  service.ts       # 该页专用接口（非 OpenAPI 生成的）
  _mock.ts         # 该页 mock，随页面一起删除
  data.d.ts        # 该页类型
  style.style.ts   # antd-style createStyles，文件名固定 *.style.ts
  components/      # 仅本页使用的子组件
  index.test.tsx   # 可选，测试就放在旁边
```

`src/pages/report/index.tsx`：

```tsx
import { PageContainer, ProTable } from '@ant-design/pro-components';
import type { ProColumns } from '@ant-design/pro-components';
import { useQuery } from '@tanstack/react-query';
import { Button } from 'antd';

import { queryReport } from './service';

export default function Report() {
  const columns: ProColumns<API.RuleListItem>[] = [
    { title: '名称', dataIndex: 'name' },
    { title: '描述', dataIndex: 'desc' },
  ];

  return (
    <PageContainer>
      <ProTable<API.RuleListItem>
        rowKey="key"
        columns={columns}
        search={{ labelWidth: 'auto' }}
        // ProTable 的 request 是「查表类页面」的默认数据入口
        request={async (params) => {
          const { data, total } = await queryReport(params);
          return { data, total, success: true };
        }}
        toolBarRender={() => [<Button key="new" type="primary">新建</Button>]}
      />
    </PageContainer>
  );
}
```

`src/pages/report/service.ts`：

```ts
import { request } from '@umijs/max';

export async function queryReport(params: { current?: number; pageSize?: number }) {
  return request<{ data: API.RuleListItem[]; total: number; success: boolean }>('/api/report', {
    method: 'GET',
    params,
  });
}
```

`src/pages/report/_mock.ts`（umi mock 约定：**default 导出** 一个「方法 + 路径 → 处理函数」的对象）：

```ts
import type { Request, Response } from 'express';

export default {
  'GET /api/report': (_req: Request, res: Response) => {
    res.json({ success: true, total: 1, data: [{ key: '1', name: '示例', desc: 'hello' }] });
  },
};
```

> `config/config.ts` 的 `mock.include` 已包含 `src/pages/**/_mock.ts`，所以页面内 `_mock.ts` 会被自动加载，与 `mock/` 下的全局 mock 并存。Mock 生效时控制台能看到对应请求，`MOCK=none` 时则不注册。

### 第 2 步：注册路由（即菜单）

`config/routes.ts` 加一项：

```ts
{
  path: '/report',
  name: 'report',        // 自动映射为国际化 key：menu.report
  icon: 'barChart',      // 图标名去风格后缀，参考 @ant-design/icons
  component: './report', // 相对 src/pages
  access: 'canAdmin',    // 可选：路由级权限，无权限则菜单不显示
},
```

`name` 支持多级菜单：`/dashboard/analysis` 的 key 是 `menu.dashboard.analysis`（点号逐级拼接）。

### 第 3 步：补国际化

`src/locales/zh-CN/menu.ts` 与 `src/locales/en-US/menu.ts` 至少各加一行（其余 6 种语言可后补，缺失时会回退显示 key）：

```ts
'menu.report': '报表',   // en-US: 'menu.report': 'Report',
```

### 第 4 步：自查

```bash
npm run biome        # 自动修复
npm run lint         # Biome + tsc
npm test             # 单测
npx antd lint ./src  # antd 用法
npm start            # 肉眼确认菜单、mock 数据、权限
```

---

## 6. 请求与数据流

- **请求实例**：`@umijs/max` 的 `request`。`baseURL` 在 `src/app.tsx` 里按环境切换（开发为空串走同源/代理，生产为 `https://pro-api.ant-design-demo.workers.dev`）。
- **错误处理**：`src/requestErrorConfig.ts`。后端约定响应体 `{ success, data, errorCode, errorMessage, showType }`；`success: false` 会被 `errorThrower` 抛出 `BizError`，再按 `showType` 决定静默 / message / notification / 跳登录。单个请求可以传 `{ skipErrorHandler: true }` 自行处理（`getInitialState` 就是这么做的）。
- **数据获取的三种选择**：
  - `ProTable` / `ProForm` 等 ProComponents 的 `request` prop —— 列表、表单页首选。
  - `@tanstack/react-query` 的 `useQuery` / `useMutation` —— 需要缓存、失效、乐观更新时（`src/pages/table-list/index.tsx` 有完整例子：删除后 `invalidateQueries` + `actionRef.current?.reloadAndRest?.()`）。
  - `useRequest`（ahooks，umi 内置）—— 简单一次性请求。
- **全局状态**：`useModel('@@initialState')` 拿 `currentUser` / `settings`；需要自己的全局 hook 时，在 `src/models/` 下新建文件即自动注册（该目录当前为空，按需创建），组件里 `useModel('文件名')`。
- **登录态**：`getInitialState()` 请求 `GET /api/currentUser`；未登录 → 重定向 `/user/login?redirect=...`。401 处理也归它。

---

## 7. 样式怎么写（按优先级从上到下选）

1. **Tailwind CSS v4** —— 布局、间距、flex/grid。入口是 `tailwind.css`（`@import "tailwindcss/index.css"` + `@source "./src"`），在 `src/global.tsx` 引入。`src/pages/Welcome.tsx` 是最完整的示例。
2. **antd-style 的 `createStyles`** —— 需要主题 token（`token.colorTextSecondary` 等）、伪类、媒体查询时。文件名固定 `*.style.ts`，默认导出 hook，组件里 `const { styles } = useStyles()`。
3. **CSS Modules / 普通 css** —— 特例场景（如 `src/pages/Welcome.css`）。
4. **Less** —— 仅历史遗留（`src/global.less`），新代码不要用。

主题色、布局模式改 `config/defaultSettings.ts`；antd token 在 `config/config.ts` 的 `antd.configProvider.theme` 里（当前设置了字体 `AlibabaSans` 和 `variant: 'filled'`）。

---

## 8. 权限与国际化

**权限**：`src/access.ts` 返回布尔映射，目前只有 `canAdmin`。用法有两种：

```ts
// 路由级：config/routes.ts
{ path: '/report', access: 'canAdmin', component: './report' }

// 组件级
import { useAccess, Access } from '@umijs/max';
const access = useAccess();
{access.canAdmin && <Button>仅管理员可见</Button>}
<Access accessible={access.canAdmin} fallback={<div>无权限</div>}>...</Access>
```

**国际化**：`useIntl().formatMessage({ id, defaultMessage })`，或组件用 `<FormattedMessage id="..." defaultMessage="..." />`。文案按模块拆在 `src/locales/<locale>/` 下的 `menu / pages / component / settings / globalHeader / network / settingDrawer` 文件里，再由 `src/locales/<locale>.ts` 汇总。默认语言 `zh-CN`，且 `baseNavigator: true`（会跟随浏览器语言）。新增语言需要补一整套 `<locale>/` 目录 + `<locale>.ts`。

---

## 9. 测试

- 运行器是 **Vitest**（`vitest.config.ts`），环境 `happy-dom`，`globals: true`（`describe/it/expect` 免 import），公共 mock 在 `tests/setupTests.ts`（localStorage、Worker、ResizeObserver、matchMedia 等已垫好）。
- 测试与源码同目录：`src/**/*.{test,spec}.{ts,tsx}`。参考 `src/pages/table-list/index.test.tsx`。
- **例外**：`src/pages/user/login/login.test.tsx` 被 `vitest.config.ts` 排除，它依赖 Umi 的测试运行时，跑不起来（同目录还有 `.snap`），别改。
- 覆盖率排除生成代码与样式：`src/.umi/**`、`src/services/ant-design-pro/**`、`*.style.ts`。

> 注意：`CLAUDE.md` 里写的 "`npm run test` (Jest)" 是旧信息，实际是 Vitest；`docs/cheatsheet.zh-CN.md` 里的 `npm run test:update` 也不存在，更新快照用 `npx vitest run -u`。

---

## 10. 提交与 CI

本地钩子：`pre-commit` → lint-staged（Biome 写回）→ `commit-msg` → commitlint（Conventional Commits）。

PR 上会跑这些检查，**建议在 push 前本地跑一遍**：

| Workflow | 检查内容 |
| --- | --- |
| `ci.yml` | `biome:lint`；`tsc`；`build`（ubuntu / windows / macOS 三平台，用 utoo） |
| `coverage.yml` | `ut test:coverage` |
| `antd-cli.yml` | `antd doctor` + `antd lint ./src` + `antd usage ./src`，结果以评论形式贴到 PR |
| `react-doctor.yml` | React Doctor 体检（`doctor.config.json` 里对示例页有豁免） |

> CI 里的构建/测试是用 **utoo**（`ut` / `ut build` / `ut test:coverage`）跑的，本地用 npm 脚本即可，两者脚本名一致。

---

## 11. 常见坑

**开发相关**

- **页面白屏 / 类型突然报 `@@/` 找不到**：删掉 `src/.umi` 重启 dev server；或跑一次 `npm run prepare`（`max setup` 生成临时类型）。
- **`npm run dev` 后接口全 404**：正常。`MOCK=none` 且 `config/proxy.ts` 的 `dev` 段是注释状态 —— 也就是 dev 环境没有代理规则，请求会打到 dev server 自己身上。想看真实数据：用 `npm run start:test`（已代理到 demo worker），或在 `config/proxy.ts` 里给 `dev` 加一条 `/api/` 规则指向自己的后端。
- **代理改了不生效**：`proxy` 只在本地 dev 生效，生产构建里无效。
- **改了 `config/*.ts` 需要重启** dev server（配置不走热更新）。
- **新增依赖导致体积变大**：先 `npm run analyze`，antd 组件请从 `antd` 具名导入，别整包引入。
- **`npm run simple` 不可逆**：它按 `config/routes.simple.ts` 替换路由并删除示例页面/依赖。执行前 `git commit` 或新建分支。

**本机环境（实测）**

- 如果 `npm run tsc` 报出一堆 `cloudflare-worker/src/**` 的 `Cannot find module 'hono'` 或 `TS2883 ... cannot be named without a reference to '.pnpm/...'`，说明这个工作区的 `node_modules` 不是用 npm 装的：
  - 根 `tsconfig.json` 的 `include` 覆盖了整个仓库，`cloudflare-worker/` 也被收进来了，而它自己的依赖（hono、wrangler）需要在 `cloudflare-worker/` 目录里单独 `npm install`；根目录没装就不会解析到。
  - 出现 `.pnpm/...` 字样则说明是 pnpm 安装的：pnpm 不做依赖提升，`@types/geojson`（`src/pages/dashboard/monitor/components/Map/index.tsx` 用到的）这种间接类型依赖解析不到，于是多出 `TS7016`/`TS2883` 报错。
  - 修复：删掉 `node_modules`（顺带清理仓库里那个未跟踪的 `pnpm-lock.yaml`），用 `npm install` 重装 —— CI 走的就是 npm 的扁平 `node_modules`，那套布局下 `npm run tsc` 是通过的。要在 worker 上做类型检查，进 `cloudflare-worker/` 执行 `npm install && npm run typecheck`。

**当前基线（本机实测，便于对照）**

- `npm run biome:lint`：通过，274 个文件 0 问题。
- `npx antd lint ./src`：通过（有 33 处 deprecated + 14 处 usage 警告，主要是示例页里已废弃的 `List`，不阻断）。
- `npm run tsc`：在 npm 安装环境下通过；在 pnpm 环境下会因上面两条报错。

---

## 12. 进阶与资料

- **演示后端**：`cloudflare-worker/` 是独立的 Hono + Workers 项目（不是 npm workspace），`wrangler dev` / `wrangler deploy`，路由在 `src/routes/`（dashboard、list、table、user、profile…），前端生产环境的 `/api` 就指向它。
- **AI 助手页面**：`src/pages/chatbot/` 基于 Ant Design X（`@ant-design/x`、`x-markdown`、`x-sdk`）+ `@tanstack/react-query`。
- **仓库自带的 AI Skills**（`.claude/skills/`）：`/pro-upgrade` 自动升级到最新 Pro 模板并保留业务代码；`/antd` 查组件 API、跑 antd lint、做版本迁移。在 ZCode/Claude Code 里直接调用即可；更新技能定义用 `npx skills add ant-design/ant-design-pro`。
- **文档**：`docs/cheatsheet.zh-CN.md`（速查，内容全面）、`docs/cheatsheet.en-US.md`、`docs/roadmap/`（版本规划）。
- **上游文档**：[Umi Max](https://umijs.org/docs/max/introduce)、[ProComponents](https://procomponents.ant.design/)、[Ant Design](https://ant.design/)、[utoopack](https://utoo.land)、[Biome](https://biomejs.dev/)。
