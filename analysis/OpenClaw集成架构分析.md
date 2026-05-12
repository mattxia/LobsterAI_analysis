# OpenClaw 集成架构分析

## 总览
LobsterAI 采用插件式架构集成 OpenClaw 引擎，与内置的 Claude 引擎共享统一的运行时接口，上层业务完全透明无感知。

## 架构图
```
┌─────────────────────────────────────────────────────────────────┐
│                        Renderer Process                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ CoworkView   │  │  IMSettings  │  │  EngineStartupOverlay│  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                │                                │
├────────────────────────────────┼────────────────────────────────┤
│                          Main Process                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    CoworkEngineRouter                     │  │
│  │  (统一路由层：自动切换 openclaw / yd_cowork 引擎)         │  │
│  └───────────────────┬───────────────────┬───────────────────┘  │
│  ┌───────────────────▼───────────────────┐ ┌───────────────────┐│
│  │      OpenClawRuntimeAdapter           │ │ ClaudeRuntimeAdapter ││
│  │  (OpenClaw 运行时适配器：实现 CoworkRuntime 接口)          │ │ (内置Claude引擎)││
│  └───────────────────┬───────────────────┘ └───────────────────┘│
│  ┌───────────────────▼───────────────────┐                      │
│  │      OpenClawEngineManager             │                      │
│  │  (引擎生命周期管理：安装/启动/停止/状态监控)              │                      │
│  └───────────────────┬───────────────────┘                      │
│  ┌───────────────────▼───────────────────┐                      │
│  │      OpenClawConfigSync                │                      │
│  │  (配置同步：将 LobsterAI 配置映射到 OpenClaw 格式)        │                      │
│  └───────────────────┬───────────────────┘                      │
│  ┌───────────────────▼───────────────────┐                      │
│  │      OpenClaw Gateway Process          │                      │
│  │  (独立子进程：运行 OpenClaw 核心引擎，WebSocket 通信)      │                      │
│  └───────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

## 核心组件说明

### 1. CoworkEngineRouter ([coworkEngineRouter.ts](file:///d:/prj/LobsterAI_analysis/src/main/libs/agentEngine/coworkEngineRouter.ts))
**作用**：
- 统一的引擎路由层，实现 CoworkRuntime 接口，对上层完全屏蔽底层引擎差异
- 根据用户配置自动调度请求到 `openclaw` 或 `yd_cowork` (内置 Claude) 引擎
- 统一处理引擎切换逻辑，停止所有活跃会话并通知上层
- 统一转发所有运行时事件（消息、权限请求、错误等）

**关键特性**：
- 会话与引擎绑定：每个会话启动时确定使用的引擎，生命周期内保持一致
- 自动容错：引擎不存在时默认 fallback 到 openclaw
- 统一的权限请求路由：自动将权限响应转发到对应的引擎

---

### 2. OpenClawRuntimeAdapter ([openclawRuntimeAdapter.ts](file:///d:/prj/LobsterAI_analysis/src/main/libs/agentEngine/openclawRuntimeAdapter.ts))
**作用**：
- OpenClaw 引擎的核心适配器，完整实现 CoworkRuntime 接口
- 负责与 OpenClaw Gateway 的 WebSocket 通信
- 管理会话状态、消息流、工具权限请求
- 处理 IM 通道会话同步（钉钉/飞书/ Telegram/QQ 等）
- 实现文本流合并、消息去重、超时 watchdog 等逻辑

**关键特性**：
- 自动重连机制：网关断开后最多重试10次，指数退避
- 多会话支持：同时管理多个并行会话
- IM 通道集成：原生支持各类 IM 平台的消息同步
- 文本流智能合并：自动处理 delta 和 snapshot 两种流模式
- 客户端超时 watchdog：避免网关无响应导致会话僵死

---

### 3. OpenClawEngineManager ([openclawEngineManager.ts](file:///d:/prj/LobsterAI_analysis/src/main/libs/openclawEngineManager.ts))
**作用**：
- OpenClaw 运行时的生命周期管理核心
- 负责运行时安装、版本校验、热修复应用
- 管理 Gateway 子进程的启动、停止、重启
- 端口自动探测、健康检查、状态上报
- 环境变量注入、代理配置传递

**生命周期状态**：
```
not_installed → installing → ready → starting → running
                                      ↓
                                    error
```

**关键特性**：
- 跨平台兼容：Windows 下使用 child_process.spawn，其他平台使用 utilityProcess.fork
- 热修复机制：自动应用 bundled 的 runtime 补丁
- 端口自动扫描：默认18789端口被占用时自动向上扫描80个端口
- 健康检查：启动后主动检测网关健康状态
- 日志管理：自动轮转网关日志到用户数据目录

---

### 4. OpenClawConfigSync ([openclawConfigSync.ts](file:///d:/prj/LobsterAI_analysis/src/main/libs/openclawConfigSync.ts))
**作用**：
- 配置同步核心：将 LobsterAI 的本地配置映射为 OpenClaw 兼容的配置格式
- 统一管理 LLM 提供商配置、工具权限、沙箱模式、技能配置
- 支持动态配置更新，网关热加载

**关键映射关系**：
| LobsterAI 配置 | OpenClaw 配置 |
|----------------|---------------|
| executionMode: auto/local/sandbox | sandbox.mode: non-main/off/all |
| apiType: anthropic/openai | provider.api: anthropic-messages/openai-completions |
| IM 平台配置 | 各 IM 适配器配置 |
| 技能启用状态 | skills.enabled |

---

## 核心交互流程

### 1. 引擎启动流程
```
1. 用户选择使用 OpenClaw 引擎
2. CoworkEngineRouter 调用 OpenClawRuntimeAdapter.startSession()
3. Adapter 调用 OpenClawEngineManager.startGateway()
4. EngineManager 检查运行时是否存在 → 应用热修复 → 生成随机 token → 探测可用端口
5. 启动 Gateway 子进程，注入环境变量（代理、token、端口等）
6. 等待网关健康检查通过 → 建立 WebSocket 连接
7. 会话启动完成，开始流式消息传输
```

### 2. 会话执行流程
```
1. 用户发送 prompt → Renderer 调用 coworkService.startSession()
2. IPC 到 Main 进程 → CoworkEngineRouter 路由到 OpenClawRuntimeAdapter
3. Adapter 构建会话参数 → 通过 WebSocket 发送到 OpenClaw Gateway
4. Gateway 执行 Agent 逻辑 → 流式返回事件（messageUpdate/toolUse/permissionRequest等）
5. Adapter 处理流事件 → 转换为统一的 CoworkRuntime 事件 → 上报到 Router
6. Router 转发到上层 → Renderer 更新 UI
7. 工具权限请求：Adapter 触发 permissionRequest 事件 → UI 弹框 → 用户确认后 Adapter 发送响应到 Gateway
8. 会话完成 → Adapter 触发 complete 事件 → 清理会话状态
```

## 关键代码片段

### 引擎启动核心逻辑 (OpenClawEngineManager.startGateway)
```typescript
private async doStartGateway(): Promise<OpenClawEngineStatus> {
  // 1. 确保运行时就绪
  const ensured = await this.ensureReady();
  if (ensured.phase !== 'ready' && ensured.phase !== 'running') return ensured;

  // 2. 检查现有进程是否健康
  if (isGatewayProcessAlive(this.gatewayProcess)) {
    const healthy = await this.isGatewayHealthy(port);
    if (healthy) return this.getStatus();
    this.stopGatewayProcess(this.gatewayProcess);
  }

  // 3. 应用热修复（非bundle模式）
  const bundlePath = path.join(runtime.root, 'gateway-bundle.mjs');
  if (!fs.existsSync(bundlePath)) {
    this.applyRuntimeHotfixes(runtime.root);
  }

  // 4. 生成认证token和端口
  const token = this.ensureGatewayToken();
  const port = await this.resolveGatewayPort();

  // 5. 启动子进程（Windows特殊处理）
  let child: GatewayProcess;
  if (process.platform === 'win32') {
    child = spawn(
      process.execPath,
      [openclawEntry, 'gateway', '--bind', 'loopback', '--port', String(port), '--token', token],
      {
        cwd: runtime.root,
        env: { ...env, ELECTRON_RUN_AS_NODE: '1' },
        stdio: ['ignore', 'pipe', 'pipe'],
        windowsHide: true,
      },
    );
  } else {
    child = utilityProcess.fork(openclawEntry, forkArgs, {
      cwd: runtime.root,
      env,
      stdio: 'pipe',
      serviceName: 'OpenClaw Gateway',
    });
  }

  // 6. 等待网关就绪
  const ready = await this.waitForGatewayReady(port, GATEWAY_BOOT_TIMEOUT_MS);
  if (!ready) {
    this.setStatus({ phase: 'error', message: 'Gateway failed to start' });
    this.stopGatewayProcess(child);
    return this.getStatus();
  }

  this.setStatus({ phase: 'running', version: runtime.version });
  return this.getStatus();
}
```

### 路由层核心逻辑 (CoworkEngineRouter.startSession)
```typescript
async startSession(sessionId: string, prompt: string, options: CoworkStartOptions = {}): Promise<void> {
  const engine = this.safeResolveEngine();
  this.sessionEngine.set(sessionId, engine);
  try {
    await this.runtimeByEngine[engine].startSession(sessionId, prompt, options);
  } catch (error) {
    this.sessionEngine.delete(sessionId);
    this.clearRequestEngineBySession(sessionId);
    throw error;
  }
}
```

## 配置说明
- **运行时路径**：开发环境 `vendor/openclaw-runtime/current`，生产环境 `resources/cfmind`
- **配置文件**：用户数据目录下 `openclaw/state/openclaw.json`
- **日志路径**：用户数据目录下 `openclaw/logs/gateway.log`
- **环境变量**：
  - `OPENCLAW_SRC`: 自定义 OpenClaw 源码路径
  - `OPENCLAW_FORCE_BUILD=1`: 强制重新构建运行时
  - `OPENCLAW_SKIP_ENSURE=1`: 跳过版本校验
- **端口**：默认 18789，自动向上扫描可用端口

## 分析时间
2026-03-27
