# Playwright 简介（可用作对外说明文案）

Playwright 是一个现代 Web 自动化测试工具，主要用来做端到端测试（E2E），也能做 UI 自动化、接口联动验证、登录态复用、截图对比等。官方把它分成两层：底层的 Playwright Library 用统一 API 控制浏览器；更常用的是带测试运行器的 Playwright Test，它内置断言、并行执行、隔离环境和丰富调试工具。

参考：
- https://playwright.dev/docs/library

---

## 核心优势

1. **多浏览器支持**：Chromium、WebKit、Firefox，还能跑 Chrome、Edge 等品牌浏览器，并支持移动设备模拟。
   - https://playwright.dev/docs/browsers
2. **Web-first 自动等待**：很多操作和断言会自动等待页面达到可操作状态，减少“元素还没出来就点了”的不稳定问题。
3. **测试能力完整，适合 CI**：自带重试、追踪（trace）、截图、视频等能力，便于排查失败原因并接入持续集成。

你可以把它理解成“比 Selenium 更贴近现代前端页面的一套自动化方案”。

---

## 推荐写法：locator + expect

Playwright 推荐用 locator 和 expect 这套方式，因为断言会自动重试，直到成功或超时，从而降低脚本脆弱性。官方也明确推荐多数 E2E 场景优先使用 `@playwright/test`，而不是只用底层 `playwright` 包。

参考：
- https://playwright.dev/docs/writing-tests

---

## 最小示例

```ts
import { test, expect } from '@playwright/test';

test('登录后进入首页', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.getByLabel('Email').fill('demo@example.com');
  await page.getByLabel('Password').fill('123456');
  await page.getByRole('button', { name: '登录' }).click();
  await expect(page).toHaveURL(/dashboard/);
});
```

上面这段里，`getByLabel`、`getByRole` 是更稳的定位方式；`toHaveURL` 会自动等待，而不是立刻判断。这个思路比直接写很多 `sleep` 更可靠。

---

## 常见使用场景

- 回归测试：发布前自动跑关键流程
- 登录/下单/支付等业务链路验证
- 多浏览器兼容性测试
- 组件页面或管理后台的自动化操作
- 截图对比、录制 trace 排查失败原因

主页：
- https://playwright.dev/

---

## 新手友好：Codegen 录制

你可以运行 `npx playwright codegen`，手动在浏览器里点点点，Playwright 会帮你生成初始测试代码和定位器，适合快速上手或搭脚手架。

参考：
- https://playwright.dev/docs/codegen-intro

---

## 入门建议（从 0 到 1）

1. 安装 Playwright
2. 用 `npx playwright codegen` 录一个简单流程
3. 把生成代码整理成可维护的测试
4. 用 `expect` 补充断言
5. 接到 CI 里跑 Chromium/Firefox/WebKit 三套环境

如果你是前端或测试同学，可以先记住一句话：

> Playwright = 浏览器自动化 + 测试框架 + 调试工具链。

它最适合现代 Web 应用，尤其是 React、Vue、Next.js 这类动态页面。
