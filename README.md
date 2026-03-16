# AgentUAC
A simple tool to private agent from dangerous operations
# AgentUAC (User Account Control for AI)

为本地 AI 智能体提供运行时安全拦截与审批层。

## 概述

AgentUAC 是一个用于拦截 AI 智能体（如 OpenClaw, AutoGPT, LangChain 应用）高危操作的运行时安全工具。当 AI 尝试执行危险操作（如读取敏感文件、执行删除命令）时，系统会暂停执行并等待用户审批。

## 核心特性

- **零侵入**: 通过 Node.js `--require` 机制自动挂载
- **同步阻塞**: 使用 `SharedArrayBuffer` + `Atomics.wait` 实现真正的线程阻塞
- **跨框架兼容**: 通过 Module require hook 拦截，不依赖特定框架
- **Worker 线程分离**: 审批逻辑在独立线程中运行，不阻塞主应用

## 危险操作拦截

### 文件系统 (fs)
- 读取敏感文件: `/etc/passwd`, `~/.ssh/id_rsa`, `~/.aws/credentials`
- 写入敏感路径
- 删除文件/目录

### 命令执行 (child_process)
- `rm -rf`, `rm -r /`
- `dd if=`, `mkfs`
- `chmod 777`, `chown -R`
- `wget|`, `curl|`
- `nc -e`, `/bin/sh -i`

## 项目结构

```
agent-uac/
├── bin/
│   └── uac.js                 # CLI 入口
├── src/
│   ├── register.ts            # Preload 入口
│   ├── config.ts              # 配置管理
│   ├── interceptors/          # 核心拦截逻辑
│   │   ├── fs.ts              # 文件系统拦截
│   │   └── child_process.ts   # 命令执行拦截
│   ├── worker/
│   │   ├── client.ts          # 主线程通信客户端
│   │   └── notify-worker.ts  # Worker 线程审批逻辑
│   └── utils/
│       └── rules.ts           # 危险规则匹配
├── test-agent.js              # 测试模拟器
└── package.json
```

## 安装

```bash
npm install
npm run build
```

## 使用方法

### 方式一: CLI 包装器 (推荐)

```bash
# 运行需要监护的 AI 程序
node bin/uac.js run "node your-ai-agent.js"

# 或使用 npm scripts
npm run test
```

### 方式二: 直接 require

```javascript
// 在你的 AI 应用入口处添加
require('./dist/register');
```

## 配置

默认危险路径和命令定义在 `src/config.ts`。可以通过创建 `~/.agent-uac/config.json` 覆盖:

```json
{
  "dangerousPaths": [
    "/etc/passwd",
    "~/.ssh"
  ],
  "dangerousCommands": [
    "rm -rf"
  ],
  "timeout": 60000
}
```

## 测试运行

```bash
npm run build
npm run test
```

预期输出:
```
[TestAgent] 📖 尝试读取 /etc/passwd...
[AgentUAC] 🚨 拦截到敏感文件读取: /etc/passwd
[AgentUAC] 🛑 主线程即将进入 Atomics.wait 休眠...
[Worker] 📨 收到审批请求: READ_FILE - /etc/passwd
[Worker] ⏳ 模拟网络请求... (2.x秒)
[Worker] 👤 用户决策: 允许/拒绝
[Worker] 🔔 通知主线程
[AgentUAC] 🟢 主线程被唤醒！waitStatus: ok, 内存值: 1/2
[TestAgent] ✅ 读取成功 / ❌ 读取被拒绝
```

## 工作原理

1. **挂载阶段**: `register.ts` 通过 `--require` 加载，安装 Module require hook
2. **拦截阶段**: 当 `fs.readFileSync` 或 `execSync` 被调用时，检测是否为危险操作
3. **阻塞阶段**: 
   - 创建 `SharedArrayBuffer`
   - 发送消息给 Worker 线程
   - 主线程调用 `Atomics.wait` 进入休眠
4. **审批阶段**: Worker 模拟网络请求，随机返回允许/拒绝
5. **唤醒阶段**: Worker 调用 `Atomics.notify` 唤醒主线程，执行放行或阻断

## 待实现

- [ ] 真实 Telegram Bot 集成
- [ ] 配置文件热加载
- [ ] 日志持久化
- [ ] 白名单规则

## License

MIT