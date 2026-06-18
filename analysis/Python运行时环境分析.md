# Python 运行时环境分析

## 概述

有道龙虾（LobsterAI）在 Windows 平台内置了完整的 Python 运行时环境，使用户无需手动安装 Python 即可使用相关功能。本文档分析 Python 运行时的获取、部署、配置和使用机制，以及 pip 访问网络的证书处理方式。

## 涉及的核心文件

| 文件 | 阶段 | 职责 |
|------|------|------|
| `scripts/setup-python-runtime.js` | 构建打包阶段 | 从 python.org 下载 embeddable Python，准备 pip，输出到 `resources/python-win/` |
| `src/main/libs/pythonRuntime.ts` | 应用运行时阶段 | 将内置运行时同步到用户可写目录，注入 PATH 环境变量，修复 pip |
| `src/main/libs/coworkUtil.ts` | 应用运行时阶段 | 构建子进程环境变量，注入 Python 路径、UTF-8 编码、系统代理等 |
| `src/main/libs/systemProxy.ts` | 应用运行时阶段 | 解析系统代理并注入 `http_proxy`/`https_proxy` 环境变量 |
| `scripts/electron-builder-hooks.cjs` | 构建打包阶段 | `beforePack` 钩子确保 Python 运行时健康检查通过 |
| `electron-builder.json` | 构建打包阶段 | 配置 `asarUnpack` 解包 npm，`extraResources` 打包资源 |

## 一、Python 运行时的获取（打包阶段）

### 1.1 运行时来源

`scripts/setup-python-runtime.js` 负责在打包前准备 Python 运行时，采用三级优先级策略获取归档：

```
① LOBSTERAI_PORTABLE_PYTHON_ARCHIVE（本地离线归档，最高优先级）
    ↓ 不可用
② resources/python-win-runtime.zip（缓存归档）
    ↓ 不可用
③ 从 python.org 下载 Python 3.11.9 embeddable（默认源）
```

**默认下载源**：
- Python embeddable: `https://www.python.org/ftp/python/3.11.9/python-3.11.9-embed-amd64.zip`
- get-pip.py 引导脚本: `https://bootstrap.pypa.io/get-pip.py`
- pip.pyz 归档: `https://bootstrap.pypa.io/pip/pip.pyz`

所有 URL 均可通过环境变量覆盖，支持企业私有镜像和离线构建。

### 1.2 Windows 主机引导流程

当无预构建归档且在 Windows 主机构建时，`bootstrapRuntimeOnWindows()` 执行以下步骤：

1. 下载 Python embeddable zip 并解压到 `resources/python-win/`
2. 修改 `._pth` 文件启用 `site-packages` 和 `import site`
3. 创建 `python3.exe` 别名（复制 `python.exe`）
4. 下载 `get-pip.py` 并运行 `python get-pip.py` 安装 pip
5. 调用 `ensurePipPayload()` 确保 pip 完整可用

### 1.3 pip 保障机制

`ensurePipPayload()` 采用三级策略确保 pip 可用：

```
① 从宿主机已安装的 Python 复制 pip 包
   ↓ 失败
② 下载 pip.pyz 并创建 shim 模块（__main__.py 通过 runpy 加载 pip.pyz）
   ↓ 失败
③ 创建 pip.cmd / pip bash 包装脚本，通过 python -m pip 调用
```

### 1.4 打包集成

`electron-builder-hooks.cjs` 的 `beforePack` 钩子在 Windows 打包前：
1. 调用 `ensurePortablePythonRuntime({ required: true })` 确保运行时就绪
2. 执行健康检查（校验 `python.exe`、`python3.exe`、pip 可执行文件、pip 模块）
3. 健康检查通过后才继续打包

Python 运行时目录 `resources/python-win/` 随安装包分发到用户机器的 `Resources/python-win/`。

## 二、Python 运行时的部署（运行时阶段）

### 2.1 两段式目录设计

| 目录 | 路径 | 可写性 | 用途 |
|------|------|--------|------|
| 内置目录 | `Resources/python-win/`（打包后） | 只读 | 随安装包分发的原始运行时 |
| 用户目录 | `userData/runtimes/python-win/` | 可写 | 实际使用的运行时，可 `pip install` |

`pythonRuntime.ts` 的 `ensurePythonRuntimeReady()` 负责将内置运行时同步到用户目录：

```
1. 检查用户目录运行时是否健康
   ├─ 健康 → 直接使用，写入 runtime.json 状态文件
   └─ 不健康 → 进入步骤 2
2. 检查内置运行时是否健康
   ├─ 不健康 → 返回错误
   └─ 健康 → 进入步骤 3
3. 同步：清空用户目录 → 递归复制内置目录 → 修正 ._pth 配置
4. 同步后健康检查 → 写入 runtime.json
```

### 2.2 ._pth 配置修正

Python embeddable 默认不启用 `site-packages`，导致 `pip install` 安装的包无法被 import。`ensureEmbedSitePackages()` 修改 `._pth` 文件：

- 启用 `import site`（取消注释或添加）
- 添加 `Lib\site-packages` 路径条目

### 2.3 运行时签名与状态追踪

`computeRuntimeSignature()` 基于必需文件的大小和修改时间计算签名，写入 `runtime.json`：

```json
{
  "syncedAt": 1718600000000,
  "sourceRoot": "Resources/python-win",
  "signature": "python.exe:12345:1718600000000|python3.exe:12345:1718600000000"
}
```

用于检测运行时是否已发生变化，避免不必要的重复同步。

### 2.4 pip 运行时修复

`ensurePythonPipReady()` 在 pip 不可用时尝试修复：

1. 先确保运行时就绪（`ensurePythonRuntimeReady()`）
2. 检查 pip 健康度
3. 若 pip 缺失，运行 `python -m ensurepip --upgrade` 引导安装
4. 验证 `python -m pip --version` 确认安装成功

## 三、环境变量注入

### 3.1 Python 路径注入

`appendPythonRuntimeToEnv()`（`pythonRuntime.ts`）将 Python 路径注入子进程环境：

```typescript
// 仅 Windows 生效
env.PATH = prependPythonRoot + Scripts + 原有PATH;
env.LOBSTERAI_PYTHON_ROOT = pythonRoot;
```

### 3.2 完整环境构建

`coworkUtil.ts` 的 `applyPackagedEnvOverrides()` 和 `getEnhancedEnv()` 构建完整的子进程环境：

| 环境变量 | 值 | 用途 |
|----------|-----|------|
| `LOBSTERAI_ELECTRON_PATH` | Electron 可执行文件路径 | Node.js 运行时（通过 `ELECTRON_RUN_AS_NODE=1`） |
| `LANG` / `LC_ALL` | `C.UTF-8` | 强制 MSYS2/git-bash UTF-8 输出 |
| `PYTHONUTF8` | `1` | Python UTF-8 模式（PEP 540） |
| `PYTHONIOENCODING` | `utf-8` | Python I/O 编码 |
| `LESSCHARSET` | `utf-8` | less/git pager UTF-8 |
| `BASH_ENV` | bash 初始化脚本路径 | 每次非交互 bash 会话切换控制台到 UTF-8 代码页 |
| `PATH` | Python 根 + Scripts + 系统目录 + 注册表 PATH | 工具查找路径 |
| `http_proxy` / `https_proxy` | 系统代理 URL | pip 等工具的网络代理 |
| Windows 系统变量 | `SystemRoot`、`windir`、`COMSPEC` 等 | 确保 Windows 系统命令可用 |

### 3.3 系统代理注入

`systemProxy.ts` 通过 Electron 的 `session.defaultSession.resolveProxy()` 解析系统代理，注入到环境变量：

```typescript
env.http_proxy = proxyUrl;
env.https_proxy = proxyUrl;
env.HTTP_PROXY = proxyUrl;
env.HTTPS_PROXY = proxyUrl;
```

pip 和 requests 库会自动读取这些环境变量使用代理。

## 四、pip 访问网络的证书处理

### 4.1 结论：不需要额外预置证书

有道龙虾**没有**做任何特殊的 CA 证书预置工作。pip 访问 HTTPS 源（如 PyPI）时，SSL 证书验证完全依赖 Python 运行时和操作系统自带的证书链。

### 4.2 分析依据

**代码中无证书相关配置**：在整个代码库中搜索 `SSL_CERT_FILE`、`REQUESTS_CA_BUNDLE`、`CURL_CA_BUNDLE`、`PIP_CERT`、`certifi`、`cacert`、`ca_bundle` 等关键词，均无命中。应用没有：
- 设置 `SSL_CERT_FILE` / `REQUESTS_CA_BUNDLE` 环境变量
- 捆绑 `certifi` 包或 `cacert.pem` 文件
- 配置 pip 的 `--cert` 参数或 `pip.ini` 中的 `cert` 配置项

**环境变量注入中无证书项**：`applyPackagedEnvOverrides()` 注入的环境变量不包含任何 SSL/CA 证书相关变量。

**代理处理仅传递 URL**：`systemProxy.ts` 仅将代理 URL 注入 `http_proxy`/`https_proxy`，完全不涉及代理服务器的 TLS 证书配置。

### 4.3 Python embeddable 的证书来源

有道龙虾使用 Python 3.11.9 embeddable 版本，该版本包含一个关键的证书文件：

```
python-win/cert/cacert.pem
```

这个文件随 Python embeddable 一起从 python.org 下载，是 **Mozilla CA 证书库的副本**（由 Python 官方维护）。pip 和 urllib 在发起 HTTPS 请求时默认使用这个文件验证服务器证书。

### 4.4 潜在问题场景

| 场景 | 原因 | 解决方式 |
|------|------|----------|
| 企业 MITM 代理 | 代理使用自签名 CA 证书拦截 HTTPS | 需用户手动设置 `SSL_CERT_FILE` 环境变量指向企业 CA 证书 |
| cacert.pem 过期 | embeddable 内置证书库未更新 | 重新打包更新 Python embeddable 版本 |
| 证书库路径丢失 | 运行时同步时未完整复制 `cert/` 目录 | `cpRecursiveSync` 会完整复制，理论不会发生 |

## 五、Node.js 运行时（对比）

与 Python 不同，有道龙虾**没有单独打包 Node.js**，而是复用 Electron 自身的二进制：

| 机制 | 实现 |
|------|------|
| 运行时来源 | Electron 可执行文件本身（`process.execPath`） |
| 运行方式 | 设置 `ELECTRON_RUN_AS_NODE=1` 环境变量，Electron 二进制可作为纯 Node.js 运行 |
| macOS 特殊处理 | 使用 `LobsterAI Helper.app` 内的 Helper 可执行文件（避免 Electron GUI 框架干扰） |
| npm/npx | 通过 `asarUnpack` 解包 `node_modules/npm/`，创建 shim 脚本调用 `npm-cli.js`/`npx-cli.js` |
| 优先级 | 优先使用系统已安装的 `node`，系统无 node 时回退到 Electron 二进制 |

## 六、整体架构流程

```
打包阶段                              运行时阶段
┌─────────────────────────┐         ┌──────────────────────────────┐
│ setup-python-runtime.js │         │ pythonRuntime.ts             │
│                         │         │                              │
│ python.org 下载          │         │ ensurePythonRuntimeReady()   │
│ embeddable Python 3.11.9│         │  内置目录 → 用户目录同步      │
│         ↓               │         │  修正 ._pth 配置              │
│ 解压到 resources/       │         │         ↓                    │
│ python-win/             │ ──────→ │ ensurePythonPipReady()       │
│         ↓               │ 打包分发 │  pip 健康检查 + ensurepip     │
│ 安装 pip (get-pip.py    │         │         ↓                    │
│   或 pip.pyz shim)      │         │ appendPythonRuntimeToEnv()   │
│         ↓               │         │  注入 PATH + LOBSTERAI_      │
│ 健康检查通过             │         │  PYTHON_ROOT                 │
└─────────────────────────┘         │         ↓                    │
                                    │ coworkUtil.ts                │
                                    │  applyPackagedEnvOverrides() │
                                    │  注入 UTF-8 编码、系统代理     │
                                    │         ↓                    │
                                    │ Claude Agent SDK 子进程       │
                                    │  python / pip 可直接使用      │
                                    └──────────────────────────────┘
```

## 七、关键设计决策总结

1. **仅 Windows 内置 Python**：macOS/Linux 依赖系统已安装的 Python，减少包体积
2. **两段式目录**：内置目录只读，同步到用户目录可写，解决 `pip install` 权限问题
3. **embeddable 而非完整安装**：体积更小，但需要手动启用 site-packages
4. **pip 多级保障**：宿主机复制 → pip.pyz 下载 → ensurepip 引导，最大化可用性
5. **无证书管理**：依赖 Python embeddable 自带的 cacert.pem，简化部署但企业代理环境需用户自行处理
6. **Node.js 复用 Electron**：不单独打包 Node.js，通过 `ELECTRON_RUN_AS_NODE=1` 复用 Electron 二进制
