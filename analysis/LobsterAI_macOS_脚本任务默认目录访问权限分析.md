# LobsterAI macOS 脚本任务默认目录访问权限分析

## 核心结论

LobsterAI 在 macOS 上运行的“脚本任务”默认**没有系统级沙箱限制**，可以访问当前 macOS 用户拥有 UNIX 权限的几乎所有目录。

---

## 1. 关键前提：未启用 macOS App Sandbox

应用打包使用的 entitlement 配置文件：

- `build/entitlements.mac.plist`

其中**没有** `com.apple.security.app-sandbox` 声明。因此子进程（脚本任务）与主进程一样，不受 macOS Sandbox 的文件访问边界限制。

同时，`electron-builder.json` 中开启了 Hardened Runtime，并配置了公证脚本 `scripts/notarize.js`。Hardened Runtime 主要限制代码签名、JIT、动态库加载等行为，并不限制文件系统访问范围。

---

## 2. OpenClaw 的“sandbox”已被强制关闭

虽然界面上存在 `auto / local / sandbox` 三种执行模式，但同步到 OpenClaw 配置时被强制映射为 `off`：

- `src/main/libs/openclawConfigSync.ts`

```ts
const mapExecutionModeToSandboxMode = (_mode: CoworkExecutionMode): 'off' | 'non-main' | 'all' => {
  // Sandbox mode disabled — always run locally
  return 'off';
};
```

无论用户选择哪种模式，OpenClaw 实际拿到的 `sandbox.mode` 都是 `'off'`。因此脚本任务始终以本地进程身份运行，没有额外的进程级隔离或 chroot。

---

## 3. Cowork 中 Bash/Shell 工具的访问范围

内置 Claude Agent SDK 引擎启动时：

- `src/main/libs/coworkRunner.ts`

```ts
const envVars = await getEnhancedEnvWithTmpdir(cwd, 'local');
```

`getEnhancedEnvWithTmpdir` 会把子进程环境变量中的 `TMPDIR`（以及 Windows 上的 `TMP`/`TEMP`）指向工作目录下的 `.cowork-temp`：

- `src/main/libs/coworkUtil.ts`

```ts
const tempDir = ensureCoworkTempDir(cwd);
env.TMPDIR = tempDir;
```

### 默认访问特征

| 项目 | 默认路径/行为 |
|------|--------------|
| 工作目录（cwd） | 用户设置的 Cowork 工作目录；未设置时 OpenClaw 默认使用 `~/.openclaw/workspace` |
| 临时目录 | 被重定向到 `<workspace>/.cowork-temp` |
| PATH / HOME | 继承自 Electron 主进程，并会解析用户 shell 的 PATH（macOS 上通过 `SHELL -lc 'echo __PATH__=$PATH'`） |
| 可访问范围 | 当前 macOS 用户有权限的任何目录 |

`runClaudeCodeLocal` 通过 Claude Agent SDK 调用 Bash 工具时，命令直接在当前 `cwd` 下由 `child_process` 派生，没有 chroot、没有沙箱目录白名单。

---

## 4. SKILL 脚本的访问范围

自定义 SKILL 执行时：

- `src/main/skillManager.ts`

```ts
const result = await runScriptWithTimeout({
  command: runtime.command,
  args: [scriptPath, ...scriptArgs],
  cwd: skillDir,
  env,
  timeoutMs,
});
```

SKILL 脚本的 `cwd` 被设为该 SKILL 的目录，通常是：

- 打包生产版：`/Applications/LobsterAI.app/Contents/Resources/SKILLs/<skill-id>/`
- 或用户数据目录：`~/Library/Application Support/LobsterAI/SKILLs/<skill-id>/`

SKILL 脚本继承 `buildSkillEnv()` 构造的环境，包含用户 shell PATH 和 `HOME`，因此同样可以自由访问用户有权限的任何目录，不会被困在 SKILL 目录内。

---

## 5. 实际默认能访问的典型目录

基于以上机制，脚本任务在 macOS 上默认可以读写：

- `~/.openclaw/workspace`（或用户自定义的 Cowork 工作目录）
- `~/Library/Application Support/LobsterAI`（应用数据、数据库、日志、SKILLs）
- `~/.openclaw/`（OpenClaw 状态、记忆文件）
- `~/Downloads` / `~/Documents` / `~/Desktop` 等普通用户目录
- `/tmp` / `$TMPDIR`（不过 Claude Agent SDK 子进程被重定向到工作区下的 `.cowork-temp`）
- 任何当前用户有 `rwx` 权限的其他目录

---

## 6. 唯一存在的限制

1. **macOS 系统完整性保护（SIP）**：`/System`、`/usr/bin`（部分）、`/bin` 等受 SIP 保护的目录即使 root 也不能随意修改，但脚本通常不需要写入这些位置。
2. **TCC 隐私权限**：只有 Calendar / Reminders / Apple Events 声明了 usage description；文件访问没有 TCC 弹窗，直接由 UNIX 权限决定。
3. **工具安全策略**：CoworkRunner 对删除类命令（`rm`、`rmdir`、`git clean` 等）有额外的 `enforceToolSafetyPolicy` 审批逻辑，但这是应用层限制，不是系统文件权限限制。

---

## 总结

在 macOS 上，LobsterAI 的脚本任务默认以当前登录用户身份、在无 App Sandbox 的环境下运行，能够访问该用户拥有读写权限的几乎所有目录。唯一受约束的是：

- SIP 保护的系统目录
- 特定 TCC 隐私 API（日历、提醒事项、Apple Events）
- 应用层对危险操作（如删除）的二次确认
