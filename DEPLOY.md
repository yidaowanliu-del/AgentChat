# AgentChat 部署指南（供 Agent 阅读执行）

> 本文档面向 AI Agent（Claude Code 等）。目标：在一台新机器上从零部署 AgentChat，
> 使 `/agentchat-oneweb` 等 skill 可用。请按顺序执行，遇到决策点按「环境判断」选择。

---

## 0. 前置事实（Agent 必读）

1. **机器差异全部走 `.env`，源码零改动**。代理、Chrome 路径、端口、状态目录都是 `.env` 配置项。
2. `.env`、`.venv/`、`.state/`、`node_modules/` 均在 `.gitignore` 中——clone 后是干净仓库，机器私有文件不会被提交。
3. 启动脚本 `scripts/start-chrome-debug.sh` 的行为：
   - 存在 `<仓库根>/.venv/bin/python` 时优先使用（系统 Python < 3.10 时必需）
   - `PROXY_SERVER` 未配置/为空 → Chrome 直连；配置了 → 走该代理并做可达性检查
   - 状态目录（PID 文件）默认 `~/.local/state/agentchat/`，可用 `AGENTCHAT_STATE_DIR` 覆盖
4. 端口冲突：若本机日常 Chrome 恰好占用 9222 的 IPv4，调试 Chrome 只能绑到 IPv6，CDP 请求会打到错误进程。**换 `CDP_PORT`（如 9333）即可**，不要让用户关闭日常浏览器。

---

## 1. 安装依赖

```bash
git clone https://github.com/yidaowanliu-del/AgentChat.git && cd AgentChat
npm install
(cd skills/agentchat-oneweb && npm install)
```

Python 依赖装进项目 venv（避免污染系统 Python；daemon 脚本要求 Python ≥ 3.10 语法）：

```bash
uv sync                             # 按 uv.lock 精确复现（Python ≥3.10 自动解析）
uv sync --group pdf                 # 可选：PDF 兜底管线（Typst 不可用时的 WeasyPrint+matplotlib）
# 无 uv 时: python3 -m venv .venv && .venv/bin/pip install playwright websocket-client
```

> 校验：`.venv/bin/python -c "import playwright, websocket; print('ok')"`
> 若用户系统 Python ≥ 3.10 且不愿用 venv，可跳过本步——脚本会回退到 `python3`。

## 2. 配置 `.env`

```bash
cp .env.example .env
```

按下表逐项设置（**这是唯一需要按机器定制的步骤**）：

| 变量 | 必填 | 怎么定 |
|------|------|--------|
| `CHROMIUM_PATH` | ✅ | 系统 Chrome 路径。macOS: `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`；Linux: `/usr/bin/google-chrome-stable`；Windows: `C:\Program Files\Google\Chrome\Application\chrome.exe`。用 `ls` 验证存在 |
| `PROXY_SERVER` | 视网络 | **能直连海外 → 整行注释掉**（脚本会直连，不再强制代理）。**需代理 → 保留** `http://127.0.0.1:7897` 并改成实际代理端口 |
| `CDP_PORT` | 可选 | 默认 9222。若被占用（见 §0.4）改为 9333 等空闲端口 |
| `AGENTCHAT_STATE_DIR` | 可选 | 仅当 `~/.local/state` 不可写（如属主为 root 且用户不愿 sudo）时设置，如 `<仓库根>/.state/agentchat` |
| `CHROME_PROFILE` | 可选 | 默认 `~/.chrome-debug-profile`，一般不动 |
| `AGENTCHAT_IMAGE_DIR` | 可选 | 图片生成下载目录。默认 `process.cwd()`（在哪运行 skill 落哪）；建议设为 `ai-images/` 避免污染仓库根（该目录已在 .gitignore） |

## 3. 启动 Chrome daemon

```bash
bash scripts/start-chrome-debug.sh
```

预期输出最后一行 `READY`。然后验证 CDP：

```bash
curl -s -m 3 "http://127.0.0.1:${CDP_PORT:-9222}/json/version"
```

返回 JSON（含 `"Browser": "Chrome/…"`）即成功。

**常见故障**（详见 README 故障排查节）：

| 症状 | 处置 |
|------|------|
| `flock: command not found` | 已兼容，会自动跳过；若仍报错说明用了旧版脚本 |
| Chrome 打开后报 `ERR_PROXY_CONNECTION_FAILED` | `.env` 的 `PROXY_SERVER` 指向了不存在的代理 → 注释掉或改对端口，重启 daemon |
| daemon 日志出现 `TypeError: unsupported operand type(s) for \|` | Python < 3.10 → 按 §1 建 venv 后重启 |
| `PermissionError: ~/.local/state/agentchat` | 见上表 `AGENTCHAT_STATE_DIR`，或让用户 `sudo chown -R $(whoami) ~/.local/state` |
| CDP 404 / 连不上但 daemon 说 READY | 端口被日常 Chrome 占用 → 换 `CDP_PORT` 重启 |
| Gemini 页面 `about:blank` | GFW/代理问题：确认 `PROXY_SERVER` 正确且代理进程在跑 |

## 4. 登录网页 AI（需要用户参与）

daemon 以 GUI 模式拉起专用 Chrome（与用户日常 Chrome 隔离，独立 profile）。
Agent 无法代替用户登录，请提示用户在弹出的窗口中按需登录：

| AI | 地址 | 备注 |
|----|------|------|
| Gemini | gemini.google.com | 需 Google 账号；Pro 订阅账号会自动启用 Pro Extended |
| ChatGPT | chatgpt.com | 未登录也有免费游客额度 |
| Claude | claude.ai | 需 Anthropic 账号 |
| Kimi | kimi.moonshot.cn | 国内直连 |
| DeepSeek | chat.deepseek.com | 国内直连 |
| Qwen | www.qianwen.com | 国内直连 |
| 豆包 | www.doubao.com/chat/ | 国内直连 |
| MiniMax / MiMo | agent.minimaxi.com / aistudio.xiaomimimo.com | 国内直连 |

登录态持久保存在 `CHROME_PROFILE`，**每家只需登录一次**，重启不丢。

## 5. 验证端到端链路

```bash
node skills/agentchat-oneweb/index.js "请用一句话介绍你自己"
```

成功标志：输出含 `✓ <Provider>: USED` 和 `[receipt] AGENTCHAT_RUN`。
失败会自动沿降级链切换 provider；全部失败时按 §3 故障表排查。

单 provider 诊断：

```bash
node skills/agentchat-oneweb/index.js --provider gemini "什么是二叉树？一句话"
```

## 6. Claude Code skill 接入

```bash
ln -s "$(pwd)/skills/agentchat-oneweb" ~/.claude/skills/agentchat-oneweb
ln -s "$(pwd)/skills/agentchat-independenttasks" ~/.claude/skills/agentchat-independenttasks
ln -s "$(pwd)/skills/agentchat-websubagent" ~/.claude/skills/agentchat-websubagent
```

链接后重启 Claude Code 会话即可通过 `/agentchat-oneweb` 等调用。

---

## 附录：两台机器的参考配置

**公司机（网络直连海外）**：
```ini
CHROMIUM_PATH=/Applications/Google Chrome.app/Contents/MacOS/Google Chrome
CDP_PORT=9333
# PROXY_SERVER 不配置
AGENTCHAT_STATE_DIR=<仓库根>/.state/agentchat
```

**家庭机（需代理）**：
```ini
CHROMIUM_PATH=<按实际路径>
CDP_PORT=9222
PROXY_SERVER=http://127.0.0.1:7897
```

> 部署完成后，Agent 应向用户确认：①daemon 状态 ②已登录哪几家 AI ③跑一次 §5 验证。
