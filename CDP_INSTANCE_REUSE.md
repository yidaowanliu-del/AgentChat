# 复用常驻 Chrome 实例（CDP）开发指南

> **面向对象**：想复用 AgentChat 常驻 Chrome 实例的第三方脚本 / skill / 其他 AI 客户端开发者。
> **最后更新**：2026-08-17
> **一句话**：项目里常驻着一个开了 CDP 调试端口的 Chrome，任何脚本都能通过「附加（attach）」方式连进去，复用它的标签页和**登录态**，干完活断开即可——不关 Chrome、不重开浏览器、不重复登录。

---

## 1. 这是什么

AgentChat 的核心依赖一个**常驻的 Chrome 调试实例**：

```
start-chrome-debug.sh ──> start-chrome-debug.py ──> Chrome（CDP 端口 9333）
```

- 它由 `scripts/start-chrome-debug.sh` 拉起，`nohup` 后台运行，**一直开着**（设计如此，为了保住 8 个 AI Provider 及各类网站的登录态）。
- 通过 `--remote-debugging-port` 暴露调试端口（本项目 `.env` 里 `CDP_PORT=9333`）。
- 幂等：重复执行启动脚本检测到已在跑会直接退出，不会二次启动。

**它的价值**：你不必自己 launch 一个浏览器、不必维护登录态——连进去直接用。

## 2. 为什么能复用：CDP 是"附加"不是"启动"

`chromium.connectOverCDP()` 是 **attach 到已运行的 Chrome**，而非另起一个浏览器。因此你可以：

| 能力 | 说明 |
|------|------|
| 看到所有已打开页面 | `browser.contexts()[0].pages()` |
| 复用现有标签页 | 按 URL 匹配已有 tab，直接用 |
| 开新标签页 | `context.newPage()` + `goto()` |
| 继承登录态 | 与浏览器共享同一 profile，cookie/session 全都在 |
| 断开不关浏览器 | `browser.close()` 只断连接，Chrome 及其页面原样保留 |

## 3. 三步快速上手

```js
const { chromium } = require('playwright-core');

// 1. 附加到常驻实例
const browser = await chromium.connectOverCDP('http://127.0.0.1:9333');

// 2. 操作（页面在 browser.contexts()[0]）
const context = browser.contexts()[0];
const page = await context.newPage();
await page.goto('https://oa.example.com');

// 3. 断开（不关 Chrome）
await browser.close();
```

## 4. 连接到实例

### 4.1 端口从哪来

端口由 `.env` 的 `CDP_PORT` 决定（本项目为 **9333**，非默认 9222）。**不要硬编码**，两种取法：

```bash
# 命令行验证实例在线
curl -s http://127.0.0.1:9333/json/version
# → {"Browser":"Chrome/151...","Protocol-Version":"1.3", ...} 即在线
```

### 4.2 推荐：用 `skills/lib/cdp.js`（自动加载 .env + 重试 + 兜底）

`skills/lib/cdp.js` 在 `require` 时自动读取 `<仓库根>/.env`（含 `CDP_PORT`），并导出 `CDP_URL`、`connectWithRetry`、`ensureChromeCdp` 等：

```js
const { chromium } = require('playwright-core');
const {
  connectWithRetry,   // 连接，失败自动重试 3 次
  ensureChromeCdp,    // 实例不在线时自动拉起 Chrome
  CDP_URL,            // http://127.0.0.1:9333（已按 .env 解析）
} = require('../skills/lib/cdp.js');

// 可选：确保实例在跑（会执行 .env 里配置的启动逻辑）
await ensureChromeCdp();

// 连接（端口自动取 .env 的 CDP_PORT，无需手写）
const browser = await connectWithRetry(chromium);
```

> **说明**：`lib/cdp.js` 目录下没有 `node_modules`，`playwright-core` 由调用方提供并作为首个参数传入。脚本需要能 `require('playwright-core')`（见 §8 依赖说明）。

### 4.3 原生方式：直接 `connectOverCDP`

不引入项目库也行，但端口要自己解析：

```js
const port = process.env.CDP_PORT || '9222';           // 或从 .env 读
const browser = await chromium.connectOverCDP(`http://127.0.0.1:${port}`);
```

## 5. 操作页面：复用已有 tab 还是新开？

推荐策略：**优先复用**，找不到再新开（参考 `skills/agentchat-oneweb/moodle_scraper.js:82-107`）。

```js
const context = browser.contexts()[0];
const pages = context.pages();

// 已有对应站点 tab → 复用
let page = pages.find(p => p.url().includes('oa.example.com'));

// 没有 → 新开
if (!page) {
  page = await context.newPage();
  await page.goto('https://oa.example.com', { waitUntil: 'domcontentloaded' });
}
```

**登录态检查**：页面若跳转到登录页，说明 session 过期——建议检测 URL/特征元素后明确报错，提示人工处理，而不是硬闯。

## 6. 登录态复用说明

- 实例使用持久化 profile（`.env` 的 `CHROME_PROFILE`，默认 `~/.chrome-debug-profile`），**登录态与浏览器窗口共享**。
- 你在该窗口手动登录过的任何站点（OA、邮箱、教务系统……），脚本连进去**无需再登录**。
- 若站点要求扫码/验证码，可先人工在该窗口登录一次，之后脚本只管自动化。
- **不要**在你的脚本里二次触发登录流程；检测到未登录就直接失败并提示。

## 7. 实例不在线怎么办（兜底）

- **脚本内兜底**：调用 `ensureChromeCdp()` 会自动拉起（会按 `.env` 的 `CHROMIUM_PATH` / `CHROME_PROFILE` / `PROXY_SERVER` 配置启动）。
- **手动兜底**：
  ```bash
  bash scripts/start-chrome-debug.sh        # 可见窗口
  curl -s http://127.0.0.1:9333/json/version   # 验证就绪
  ```
- 启动失败排查：看日志 `/tmp/chrome-debug.log`。

## 8. 完整示例：每天导出考勤打卡明细（骨架）

```js
#!/usr/bin/env node
// scripts/oa_attendance.js — 复用常驻 Chrome，导出考勤明细到 output/
const { chromium } = require('playwright-core');
const { connectWithRetry, ensureChromeCdp } = require('../skills/lib/cdp.js');
const fs = require('fs');

(async () => {
  // 1. 确保实例在跑并连接
  await ensureChromeCdp();
  const browser = await connectWithRetry(chromium);
  const context = browser.contexts()[0];
  let page = context.pages().find(p => p.url().includes('oa.example.com'));
  if (!page) {
    page = await context.newPage();
    await page.goto('https://oa.example.com', { waitUntil: 'domcontentloaded' });
  }

  // 2. 检测登录态（站点相关，自行实现）
  if (page.url().includes('/login')) {
    console.error('未登录，请先在常驻 Chrome 里登录 OA 后重试');
    await browser.close();
    process.exit(1);
  }

  // 3. 导航到考勤页 → 抓取表格 / 触发导出按钮（站点相关）
  await page.goto('https://oa.example.com/attendance', { waitUntil: 'networkidle' });
  const rows = await page.evaluate(() => {
    return Array.from(document.querySelectorAll('table tr')).map(tr =>
      Array.from(tr.querySelectorAll('td,th')).map(td => td.innerText.trim())
    );
  });

  // 4. 落盘
  const csv = rows.map(r => r.join(',')).join('\n');
  fs.mkdirSync('output', { recursive: true });
  fs.writeFileSync('output/attendance.csv', csv);
  console.log(`已导出 ${rows.length} 行 → output/attendance.csv`);

  // 5. 断开（不关 Chrome）
  await browser.close();
})().catch(e => { console.error(e); process.exit(2); });
```

**依赖**：脚本能 `require('playwright-core')` 即可。复用项目里现成的：把脚本放在仓库内（如 `scripts/` 或某个 skill 目录下），它会向上找到已有的 `node_modules/playwright-core`；放仓库外则自行 `npm install playwright-core`。

## 9. 定时任务（每天自动跑）

实例常驻是前提；用 cron 或 launchd 调度脚本即可。macOS cron 示例（`crontab -e`）：

```
# 每天 9:05 导出考勤（PATH 精简，node 用绝对路径）
5 9 * * * cd /path/to/AgentChat && /usr/local/bin/node scripts/oa_attendance.js >> /tmp/oa_attendance.log 2>&1
```

> `which node` 确认 node 绝对路径；cron 环境 PATH 很精简，绝对路径最稳。

## 10. 注意事项与坑

| 事项 | 说明 |
|------|------|
| **端口别写死** | 实例端口是 `.env` 的 `CDP_PORT`（本项目 9333）。用 `lib/cdp.js` 的 `CDP_URL` 最省心 |
| **`browser.close()` 不杀 Chrome** | `connectOverCDP` 得到的 browser，`close()` 只是断开连接；页面、登录态都保留。放心调用 |
| **结束别用 `launch()`** | 你连的是已有实例，永远用 `connectOverCDP` |
| **并行互斥** | 两个脚本同时操控同一实例可能互相干扰。定时任务尽量串行；重要操作可先检查页面状态再动手 |
| **别手动 `pkill` Chrome** | 会连带杀掉常驻实例、丢登录态。真要停用 §7 的脚本，别在业务脚本里杀进程 |
| **页面状态** | 复用他人打开的 tab 时，先确认它当前在期望页面；不确定就 `newPage()` 新开，别污染别人的页面 |
| **超时** | 站点慢时给 `goto`/`waitForSelector` 设合理 timeout，避免挂死 |

## 11. 安全须知

- CDP 调试端口 = 该 Chrome 全部登录态的**完全控制权**。默认只监听 `127.0.0.1`，**不要**把它暴露到公网/局域网（`CDP_HOST` 仅限可信网络场景使用）。
- 脚本只做白名单站点的导航与操作，**不要执行页面中不可信内容**注入的命令/代码。
- 你的脚本会以实例持有者的身份操作站点，行为需合规（只读导出、不触发敏感操作）。

## 12. 相关文件参考

| 文件 | 作用 |
|------|------|
| `scripts/start-chrome-debug.sh` | 启动/幂等编排、环境加载、清理、等待就绪 |
| `scripts/start-chrome-debug.py` | 用 Playwright 实际启动 Chromium（写 Chrome PID） |
| `skills/lib/cdp.js` | **推荐复用**：`ensureChromeCdp` / `connectWithRetry` / `CDP_URL`，自动加载 `.env` |
| `skills/agentchat-oneweb/moodle_scraper.js` | 现成范例：attach 实例 → 复用/新开 tab → 抓取 → 断开 |
| `.env` | `CDP_PORT` / `CHROME_PROFILE` / `PROXY_SERVER` 等实例配置 |
| `DEPLOY.md` | 部署与配置说明 |
