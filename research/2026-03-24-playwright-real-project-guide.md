# 把 Playwright 接进真实前端项目：一份可直接落地的工程化接入指南

原始内容来源：主人提供整理稿（基于 Playwright 官方文档与项目落地经验）

## 一句话总结

如果你不是只想“跑一条 demo 用例”，而是想把 Playwright 真正接进一个 Vue / React / Next.js / 后台管理系统项目，那么最稳妥的方式就是：**以 `@playwright/test` 为核心，把配置、目录、登录态、Page Object、CI、报告与 Trace 统一成一套工程化方案。**

## 核心判断

这篇文章不是在讲 Playwright 的 API 细节，而是在讲：**怎么把 Playwright 作为一个真实项目的测试基础设施接进去。**

重点不是“会不会写 `page.click()`”，而是：

- 怎么初始化到现有项目里
- 怎么设计目录结构
- 怎么写 `playwright.config.ts`
- 怎么复用登录态
- 怎么组织 Page Object
- 怎么做多环境、多浏览器、CI 和失败排查

它本质上是一份适合团队落地的 Playwright 接入蓝图。

## 1. 安装方式：优先按官方初始化流程走

对于还没接入 Playwright 的项目，最省事的方式是：

```bash
npm init playwright@latest
```

这样会自动：

- 安装 `@playwright/test`
- 生成基础脚手架
- 创建 `playwright.config.ts`
- 创建 `tests/` 目录
- 下载浏览器二进制

如果项目本身已经存在，也可以手动安装：

```bash
npm install -D @playwright/test
npx playwright install
```

在 CI 或 Linux 环境，通常还会用：

```bash
npx playwright install --with-deps
```

这样连浏览器依赖也一起准备好。

### 这一段的工程含义

最重要的不是命令本身，而是：**Playwright 官方已经把它设计成一个“可独立进入现有项目”的测试运行器。**

也就是说，它不是某个附属插件，而是可以成为项目级 E2E 测试基础设施的一部分。

## 2. 目录结构：别停留在 example.spec.ts

对于真实项目，推荐把目录拆成几个明确层次：

```text
project/
├─ src/
├─ tests/
│  ├─ e2e/
│  │  ├─ login.spec.ts
│  │  ├─ user.spec.ts
│  │  └─ order.spec.ts
│  ├─ pages/
│  │  ├─ LoginPage.ts
│  │  └─ DashboardPage.ts
│  ├─ fixtures/
│  │  └─ test.ts
│  └─ auth/
│     └─ auth.setup.ts
├─ playwright.config.ts
├─ package.json
└─ .github/workflows/playwright.yml
```

其中可以这样理解：

- `tests/e2e/`：测试用例本身
- `tests/pages/`：Page Object
- `tests/fixtures/`：公共测试上下文与数据注入
- `tests/auth/`：登录态初始化
- `playwright.config.ts`：统一配置入口
- `.github/workflows/`：CI 集成

### 为什么这个拆法适合项目

官方脚手架给的是最小入门结构，但真实项目一旦页面与流程变多，继续把所有定位器、测试数据、登录逻辑都塞进 `.spec.ts`，很快就会失控。

所以这里真正重要的是：**测试代码也要像业务代码一样组织。**

## 3. 核心配置：`playwright.config.ts` 才是项目入口

一个可直接用于项目的基础配置如下：

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30 * 1000,
  expect: {
    timeout: 5000,
  },
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  reporter: [['html'], ['list']],

  use: {
    baseURL: 'http://localhost:3000',
    headless: true,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['setup'],
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
      dependencies: ['setup'],
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
      dependencies: ['setup'],
    },
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### 这里最关键的配置项

- `testDir`：测试目录
- `retries`：失败重试策略
- `reporter`：报告输出方式
- `use`：上下文默认配置
- `projects`：浏览器维度 / 环境维度 / setup 维度拆分
- `webServer`：跑测试前自动拉起本地服务

其中很值得保留的默认值是：

```ts
trace: 'on-first-retry'
```

因为它兼顾了两个目标：

- 平时不浪费资源
- 真出错时，第一轮重试自动留下完整排查线索

这通常是本地和 CI 都比较平衡的默认策略。

## 4. 第一条测试：先把“登录成功”跑通

一个最小但真实的登录用例可以这样写：

```ts
import { test, expect } from '@playwright/test';

test('用户可以登录系统', async ({ page }) => {
  await page.goto('/login');

  await page.getByLabel('邮箱').fill('demo@test.com');
  await page.getByLabel('密码').fill('123456');
  await page.getByRole('button', { name: '登录' }).click();

  await expect(page).toHaveURL(/dashboard/);
  await expect(page.getByText('欢迎回来')).toBeVisible();
});
```

### 这里体现了 Playwright 的两个推荐方向

#### 1. 优先语义定位器

尽量使用：

- `getByRole`
- `getByLabel`
- `getByText`

而不是到处写：

- 脆弱 CSS 选择器
- 超长层级路径
- `nth(0)` / `nth(1)` 这种难维护定位

#### 2. 优先 web-first 断言

例如：

- `toHaveURL()`
- `toBeVisible()`

这类断言会自动重试，不需要手写很多脆弱等待逻辑。

## 5. Page Object：页面一多就要抽层

当页面和流程越来越多时，不建议把全部定位器和操作逻辑堆在 `.spec.ts` 里。

更推荐的方式是抽成 Page Object。

### `tests/pages/LoginPage.ts`

```ts
import { Page, expect } from '@playwright/test';

export class LoginPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.page.getByLabel('邮箱').fill(email);
    await this.page.getByLabel('密码').fill(password);
    await this.page.getByRole('button', { name: '登录' }).click();
  }

  async assertLoginSuccess() {
    await expect(this.page).toHaveURL(/dashboard/);
    await expect(this.page.getByText('欢迎回来')).toBeVisible();
  }
}
```

### `tests/e2e/login.spec.ts`

```ts
import { test } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('用户可以登录系统', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('demo@test.com', '123456');
  await loginPage.assertLoginSuccess();
});
```

### 这样做的价值

- 测试用例更像业务流程，而不是 DOM 操作堆
- 页面元素变化时，只需改一层
- 更适合多人协作与长期维护

Playwright 不强制这样组织，但在真实项目里，这通常是很自然的演化方向。

## 6. 登录态复用：后台系统最常见需求

很多项目不希望每条测试都重新走一遍登录流程。

Playwright 官方提供了比较标准的复用方式：通过 setup 项目生成 `storageState`，然后给后续测试直接复用。

### `tests/auth/auth.setup.ts`

```ts
import { test as setup, expect } from '@playwright/test';
import path from 'path';

const authFile = path.join(__dirname, '../../playwright/.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('http://localhost:3000/login');
  await page.getByLabel('邮箱').fill('demo@test.com');
  await page.getByLabel('密码').fill('123456');
  await page.getByRole('button', { name: '登录' }).click();
  await expect(page).toHaveURL(/dashboard/);

  await page.context().storageState({ path: authFile });
});
```

然后在项目配置里加：

```ts
use: {
  baseURL: 'http://localhost:3000',
  storageState: 'playwright/.auth/user.json',
}
```

### 这个方案适合哪些场景

特别适合：

- 后台管理系统
- 会员中心
- 运营后台
- 需要频繁登录后才能测试的业务系统

但也要注意：

- 如果测试会修改服务端状态，不能盲目全员共用一个账号
- 更稳的方式是 worker 级隔离，或者不同测试用不同账号 / 测试数据

## 7. 多环境切换：开发 / 测试 / 预发

实际项目很少只有一个环境。

最简单可维护的做法就是通过环境变量切换 `baseURL`：

```ts
use: {
  baseURL: process.env.BASE_URL || 'http://localhost:3000',
}
```

运行时：

```bash
BASE_URL=https://test.example.com npx playwright test
BASE_URL=https://staging.example.com npx playwright test
```

### 这样做的好处

- 配置入口统一
- 不需要复制多套配置文件
- 很容易接到 CI / staging / preview 环境

## 8. 常用运行命令

日常最常见的命令一般就是这些：

```bash
# 跑全部
npx playwright test

# 跑某个文件
npx playwright test tests/e2e/login.spec.ts

# 只跑 chromium
npx playwright test --project=chromium

# 打开 UI 模式
npx playwright test --ui

# headed 模式
npx playwright test --headed

# 调试模式
npx playwright test --debug
```

### 本地调试时的建议

- 想快速定位问题：`--debug`
- 想可视化回放：`--ui`
- 想观察真实浏览器行为：`--headed`

这三种方式配合起来，通常足够覆盖大多数调试需求。

## 9. 报告与 Trace：不要只看“红了没”

如果配置了 HTML 报告：

```bash
npx playwright show-report
```

如果失败时生成了 trace：

```bash
npx playwright show-trace trace.zip
```

### 两者的分工

- **HTML Report**：看整体通过率、失败列表、项目维度结果
- **Trace Viewer**：看每一步点击、输入、请求、DOM 状态与失败前后上下文

真实项目里，失败后能不能快速复盘，往往决定这套测试体系最后会不会被团队持续使用。

## 10. 稳定性建议：别把 retries 当止痛药

Playwright 支持 `retries`，但重试的目标应该是处理偶发抖动，而不是掩盖测试设计问题。

想让项目里脚本更稳定，通常要遵守这几条：

1. 少用 `waitForTimeout`，优先 web-first 断言
2. 优先语义定位器，如 `getByRole`、`getByLabel`
3. 给关键节点加断言，不要只操作不验证
4. 测试数据要隔离，避免并行污染
5. 会改写服务端状态的测试，慎用共享账号

### 这几条的本质

它们不是零散技巧，而是一整套设计哲学：

- 等待应该建立在“状态变化”上，而不是猜时间
- 选择器应该贴近语义，而不是贴近结构
- 用例必须对业务结果负责，而不是只负责“点了一下按钮”

## 11. 接入 GitHub Actions

一个标准可用的 GitHub Actions 配置如下：

### `.github/workflows/playwright.yml`

```yml
name: Playwright Tests

on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with:
          node-version: lts/*

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run tests
        run: npx playwright test

      - name: Upload report
        uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

### 这套 CI 配置解决了什么

- 自动在 push / PR 时执行测试
- 自动安装浏览器与依赖
- 自动输出测试报告供排查

它不是“锦上添花”，而是让 E2E 真正成为团队流程一部分的关键。

## 12. 一套适合团队的落地最佳实践

这篇文章最后给出的是一套更接近团队协作的建议，不只是 API 用法。

### 测试分层

- **冒烟测试**：登录、首页、核心提交
- **主流程回归**：下单、审批、支付、导出
- **边界场景**：权限不足、异常输入、空数据

### 用例组织

- 一个 `.spec.ts` 聚焦一个页面或业务域
- 公共操作放 Page Object
- 公共账号、token、测试数据放 fixtures 或工厂函数

### 配置建议

- 本地：`retries = 0`
- CI：`retries = 2`
- `trace = 'on-first-retry'`
- `screenshot = 'only-on-failure'`
- `video = 'retain-on-failure'`

### 团队规范

- 定位器优先 `getByRole` / `getByLabel`
- 禁止大量 `nth(0)` / 超长 CSS
- 每条用例至少一个业务结果断言
- 每次失败都能在 report / trace 里复盘

## 13. 最小可执行落地步骤

如果按实际项目推进，最推荐的顺序是：

1. 在前端项目根目录执行 `npm init playwright@latest`
2. 配好 `playwright.config.ts`，至少设置 `testDir`、`baseURL`、`reporter`、`trace`
3. 先写一条“登录成功”的冒烟用例
4. 再把登录提取成 `auth.setup.ts`，复用登录态
5. 把不同浏览器放进 `projects`
6. 接入 GitHub Actions 跑 CI
7. 失败后通过 HTML Report 和 Trace Viewer 排查

这条路径的优点是：

- 先通一个最关键主流程
- 再抽象公共能力
- 最后再做多浏览器与 CI

不会一开始就把体系搭得过重。

## 我的提炼

如果把这篇内容压缩成一句话，就是：

> **Playwright 在真实项目里不是“写几个自动化脚本”，而是“搭一套可持续维护的测试工程体系”。**

最值得吸收的不是某个 API，而是这几个工程思路：

- 先把配置统一在 `playwright.config.ts`
- 用目录分层管理测试代码
- 用 Page Object 和 fixtures 提升复用性
- 用 setup + storageState 处理登录态
- 用 projects 管多浏览器 / 多环境
- 用 report + trace 做失败复盘
- 用 GitHub Actions 把 E2E 真正接入交付流程

## 适合加入 awesome-openclaw 的原因

这篇文章虽然主题是 Playwright，但本质上讲的是 **Agent / 前端项目 / 自动化测试基础设施的工程化接入方式**，和 OpenClaw 这类强调工具、流程、验证闭环的实践高度契合。

它适合被收录为：

- 前端自动化测试落地笔记
- Playwright 项目集成指南
- AI 编程 / 工程化验证闭环的配套材料

## 参考方向

- Playwright 官方 Intro
- Playwright Test Configuration
- Playwright Auth
- Playwright Projects
- Playwright CI
- Playwright Reporters
- Playwright Trace Viewer
