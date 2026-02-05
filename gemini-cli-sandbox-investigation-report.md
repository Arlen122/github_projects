# Gemini CLI 沙箱机制深度调查报告

## 概述

本报告详细分析 Gemini CLI 的 Docker 沙箱实现机制，重点说明主机与容器之间的关系以及关键代码位置。

---

## 一、架构总览

### 1.1 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              主机 (Host)                                     │
│                                                                              │
│   用户执行: $ gemini -s -p "分析代码"                                         │
│                    │                                                         │
│                    ▼                                                         │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │           主机上的 Gemini CLI (启动器角色)                           │    │
│   │                                                                     │    │
│   │   1. 解析命令行参数                                                  │    │
│   │   2. 加载配置和设置                                                  │    │
│   │   3. 检测沙箱配置 (GEMINI_SANDBOX / -s 标志)                         │    │
│   │   4. 完成 OAuth 认证 (重要: 在进入容器前完成)                         │    │
│   │   5. 执行 docker run 启动容器                                        │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                    │                                                         │
│                    │ docker run -i --rm --init ...                          │
│                    ▼                                                         │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │                      Docker 容器                                    │    │
│   │                                                                     │    │
│   │   环境变量: SANDBOX=gemini-cli-sandbox-0                            │    │
│   │                                                                     │    │
│   │   ┌──────────────────────────────────────────────────────────┐     │    │
│   │   │         容器内的 Gemini CLI (实际工作者)                   │     │    │
│   │   │                                                           │     │    │
│   │   │   - 与 Gemini API 进行 AI 对话                            │     │    │
│   │   │   - 执行所有 Shell 命令                                    │     │    │
│   │   │   - 进行文件读写操作                                       │     │    │
│   │   │   - 使用各种工具 (grep, git, etc.)                         │     │    │
│   │   └──────────────────────────────────────────────────────────┘     │    │
│   │                                                                     │    │
│   │   挂载卷:                                                           │    │
│   │   ├── 项目目录 (读写)                                               │    │
│   │   ├── ~/.gemini 配置目录 (读写)                                     │    │
│   │   ├── /tmp 临时目录 (读写)                                          │    │
│   │   └── ~/.config/gcloud (只读)                                       │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心问题解答

| 问题 | 答案 |
|------|------|
| 用户在用哪个 CLI? | **两者都有**: 主机 CLI 作为启动器，容器内 CLI 执行实际工作 |
| Shell 命令在哪执行? | **容器内** |
| 文件操作在哪执行? | **容器内**，通过卷挂载同步到主机 |
| API 调用在哪进行? | **容器内** |
| 认证在哪完成? | **主机上**，进入容器前完成 |

---

## 二、主机与容器的关系详解

### 2.1 主机 CLI 的职责 (启动器)

主机上的 CLI 仅负责：
1. 解析参数和配置
2. 完成认证（OAuth 需要浏览器，必须在主机完成）
3. 构建 Docker 命令并启动容器
4. 等待容器退出

**关键代码位置**: `packages/cli/src/gemini.tsx`

```typescript
// 第 446-502 行: 沙箱启动逻辑
// hop into sandbox if we are outside and sandboxing is enabled
if (!process.env['SANDBOX']) {
  const memoryArgs = settings.merged.advanced.autoConfigureMemory
    ? getNodeMemoryArgs(isDebugMode)
    : [];
  const sandboxConfig = await loadSandboxConfig(settings.merged, argv);
  
  if (sandboxConfig) {
    // 认证失败则退出
    if (initialAuthFailed) {
      await runExitCleanup();
      process.exit(ExitCodes.FATAL_AUTHENTICATION_ERROR);
    }
    
    // 读取 stdin (如果有管道输入)
    let stdinData = '';
    if (!process.stdin.isTTY) {
      stdinData = await readStdin();
    }
    
    // 注入 stdin 到参数
    const sandboxArgs = injectStdinIntoArgs(process.argv, stdinData);
    
    // 启动沙箱并等待退出
    await relaunchOnExitCode(() =>
      start_sandbox(sandboxConfig, memoryArgs, partialConfig, sandboxArgs),
    );
    await runExitCleanup();
    process.exit(ExitCodes.SUCCESS);
  }
}
```

### 2.2 容器内 CLI 的职责 (工作者)

容器内的 CLI 是完整的 Gemini CLI，负责所有实际工作：
- AI 对话和推理
- 工具调用和执行
- 文件读写
- Shell 命令执行

**判断是否在容器内的关键代码**: `packages/cli/src/config/sandboxConfig.ts`

```typescript
// 第 36-43 行
function getSandboxCommand(...): SandboxConfig['command'] | '' {
  // 如果 SANDBOX 环境变量已设置，说明已经在沙箱内
  if (process.env['SANDBOX']) {
    return '';  // 返回空，不再启动新沙箱
  }
  // ... 继续判断是否需要启动沙箱
}
```

### 2.3 两个 CLI 之间的交互

```
主机 CLI                                容器内 CLI
   │                                        │
   │  1. 准备参数和环境变量                    │
   │──────────────────────────────────────▶│
   │     (通过 docker run 参数传递)           │
   │                                        │
   │  2. 挂载卷                              │
   │◀─────────────────────────────────────▶│
   │     (双向文件同步)                       │
   │                                        │
   │  3. 传递 stdin                          │
   │──────────────────────────────────────▶│
   │     (通过 -i 参数 + 管道)                │
   │                                        │
   │  4. 接收 stdout/stderr                  │
   │◀──────────────────────────────────────│
   │     (通过 stdio: 'inherit')             │
   │                                        │
   │  5. 接收退出码                           │
   │◀──────────────────────────────────────│
   │     (容器退出时返回)                     │
```

---

## 三、关键代码文件和位置

### 3.1 核心沙箱文件

| 文件路径 | 主要功能 |
|----------|----------|
| `packages/cli/src/utils/sandbox.ts` | 沙箱启动核心逻辑，构建 docker 命令 |
| `packages/cli/src/utils/sandboxUtils.ts` | 沙箱工具函数，路径转换，入口点生成 |
| `packages/cli/src/config/sandboxConfig.ts` | 沙箱配置加载和命令检测 |
| `packages/cli/src/gemini.tsx` | CLI 入口，决定是否进入沙箱 |
| `Dockerfile` | 沙箱容器镜像定义 |

### 3.2 `sandbox.ts` 关键代码段

#### 3.2.1 启动沙箱入口 (第 38-43 行)

```typescript
export async function start_sandbox(
  config: SandboxConfig,
  nodeArgs: string[] = [],
  cliConfig?: Config,
  cliArgs: string[] = [],
): Promise<number> {
```

#### 3.2.2 构建 Docker 运行参数 (第 266-288 行)

```typescript
// 使用交互模式，退出时自动删除容器，使用 init 处理信号
const args = ['run', '-i', '--rm', '--init', '--workdir', containerWorkdir];

// 如果 stdin 是 TTY，添加 -t 参数
if (process.stdin.isTTY) {
  args.push('-t');
}

// 允许访问 host.docker.internal
args.push('--add-host', 'host.docker.internal:host-gateway');

// 挂载当前目录作为工作目录
args.push('--volume', `${workdir}:${containerWorkdir}`);
```

#### 3.2.3 挂载用户配置目录 (第 289-312 行)

```typescript
// 挂载用户设置目录到容器内
const userHomeDirOnHost = homedir();
const userSettingsDirInSandbox = getContainerPath(`/home/node/${GEMINI_DIR}`);

// 确保目录存在
if (!fs.existsSync(userSettingsDirOnHost)) {
  fs.mkdirSync(userSettingsDirOnHost, { recursive: true });
}
const userSettingsDirOnHost = path.join(userHomeDirOnHost, GEMINI_DIR);
if (!fs.existsSync(userSettingsDirOnHost)) {
  fs.mkdirSync(userSettingsDirOnHost, { recursive: true });
}

args.push('--volume', `${userSettingsDirOnHost}:${userSettingsDirInSandbox}`);
```

#### 3.2.4 挂载临时目录和 gcloud 配置 (第 314-332 行)

```typescript
// 挂载 os.tmpdir()
args.push('--volume', `${os.tmpdir()}:${getContainerPath(os.tmpdir())}`);

// 挂载 gcloud 配置目录 (只读)
const gcloudConfigDir = path.join(homedir(), '.config', 'gcloud');
if (fs.existsSync(gcloudConfigDir)) {
  args.push('--volume', `${gcloudConfigDir}:${getContainerPath(gcloudConfigDir)}:ro`);
}
```

#### 3.2.5 传递环境变量 (第 456-520 行)

```typescript
// 传递 API 密钥
if (process.env['GEMINI_API_KEY']) {
  args.push('--env', `GEMINI_API_KEY=${process.env['GEMINI_API_KEY']}`);
}

// 传递终端设置
if (process.env['TERM']) {
  args.push('--env', `TERM=${process.env['TERM']}`);
}

// 设置 SANDBOX 环境变量标识
args.push('--env', `SANDBOX=${containerName}`);
```

#### 3.2.6 Linux UID/GID 处理 (第 593-637 行)

```typescript
// 在 Linux 上处理用户权限
if (await shouldUseCurrentUserInSandbox()) {
  // 容器以 root 启动
  args.push('--user', 'root');
  
  const uid = (await execAsync('id -u')).stdout.trim();
  const gid = (await execAsync('id -g')).stdout.trim();
  
  // 创建与主机相同 UID/GID 的用户
  const setupUserCommands = [
    `groupadd -f -g ${gid} ${username}`,
    `id -u ${username} &>/dev/null || useradd -o -u ${uid} -g ${gid} -d ${homeDir} -s /bin/bash ${username}`,
  ].join(' && ');
  
  // 使用 su 切换用户执行
  const suCommand = `su -p ${username} -c '${escapedOriginalCommand}'`;
  finalEntrypoint[2] = `${setupUserCommands} && ${suCommand}`;
}
```

#### 3.2.7 启动容器进程 (第 692-713 行)

```typescript
// 暂停 stdin，启动容器进程
process.stdin.pause();
sandboxProcess = spawn(config.command, args, {
  stdio: 'inherit',  // 继承主机的 stdio
});

return await new Promise<number>((resolve, reject) => {
  sandboxProcess.on('error', (err) => {
    coreEvents.emitFeedback('error', 'Sandbox process error', err);
    reject(err);
  });

  sandboxProcess?.on('close', (code, signal) => {
    process.stdin.resume();
    resolve(code ?? 1);
  });
});
```

### 3.3 `sandboxUtils.ts` 关键代码段

#### 3.3.1 路径转换 (第 25-36 行)

```typescript
// 将主机路径转换为容器路径 (主要处理 Windows)
export function getContainerPath(hostPath: string): string {
  if (os.platform() !== 'win32') {
    return hostPath;  // Linux/macOS 直接使用相同路径
  }
  // Windows 路径转换: C:\foo\bar -> /c/foo/bar
  const withForwardSlashes = hostPath.replace(/\\/g, '/');
  const match = withForwardSlashes.match(/^([A-Z]):\/(.*)/i);
  if (match) {
    return `/${match[1].toLowerCase()}/${match[2]}`;
  }
  return withForwardSlashes;
}
```

#### 3.3.2 Linux 自动 UID/GID 检测 (第 38-72 行)

```typescript
export async function shouldUseCurrentUserInSandbox(): Promise<boolean> {
  const envVar = process.env['SANDBOX_SET_UID_GID']?.toLowerCase().trim();
  
  // 显式设置优先
  if (envVar === '1' || envVar === 'true') return true;
  if (envVar === '0' || envVar === 'false') return false;
  
  // 自动检测 Debian/Ubuntu
  if (os.platform() === 'linux') {
    const osReleaseContent = await readFile('/etc/os-release', 'utf8');
    if (osReleaseContent.includes('ID=debian') ||
        osReleaseContent.includes('ID=ubuntu') ||
        osReleaseContent.match(/^ID_LIKE=.*debian.*/m)) {
      return true;  // Debian/Ubuntu 系自动启用
    }
  }
  return false;
}
```

#### 3.3.3 容器入口点生成 (第 87-150 行)

```typescript
export function entrypoint(workdir: string, cliArgs: string[]): string[] {
  const shellCmds = [];
  
  // 1. 添加项目路径到 PATH
  if (pathSuffix) {
    shellCmds.push(`export PATH="$PATH${pathSuffix}";`);
  }
  
  // 2. 加载项目自定义 bashrc
  const projectSandboxBashrc = `${GEMINI_DIR}/sandbox.bashrc`;
  if (fs.existsSync(projectSandboxBashrc)) {
    shellCmds.push(`source ${getContainerPath(projectSandboxBashrc)};`);
  }
  
  // 3. 端口转发 (使用 socat)
  ports().forEach((p) =>
    shellCmds.push(
      `socat TCP4-LISTEN:${p},bind=$(hostname -i),fork,reuseaddr TCP4:127.0.0.1:${p} 2> /dev/null &`,
    ),
  );
  
  // 4. 构建 gemini 命令
  const cliCmd = process.env['NODE_ENV'] === 'development'
    ? 'npm run start --'
    : 'gemini';
  
  return ['bash', '-c', args.join(' ')];
}
```

### 3.4 `sandboxConfig.ts` 关键代码段

#### 3.4.1 沙箱命令检测 (第 36-94 行)

```typescript
function getSandboxCommand(sandbox?: boolean | string | null): SandboxConfig['command'] | '' {
  // 已在沙箱内，不再启动
  if (process.env['SANDBOX']) {
    return '';
  }
  
  // 环境变量优先级最高
  const environmentConfiguredSandbox = process.env['GEMINI_SANDBOX']?.toLowerCase().trim() ?? '';
  sandbox = environmentConfiguredSandbox?.length > 0 ? environmentConfiguredSandbox : sandbox;
  
  // 字符串值处理
  if (sandbox === '1' || sandbox === 'true') sandbox = true;
  else if (sandbox === '0' || sandbox === 'false' || !sandbox) sandbox = false;
  
  if (sandbox === false) return '';
  
  // 指定了具体命令
  if (typeof sandbox === 'string' && sandbox) {
    if (!isSandboxCommand(sandbox)) {
      throw new FatalSandboxError(`Invalid sandbox command '${sandbox}'`);
    }
    if (commandExists.sync(sandbox)) {
      return sandbox;
    }
    throw new FatalSandboxError(`Missing sandbox command '${sandbox}'`);
  }
  
  // 自动检测: macOS 优先 seatbelt，其他需显式启用
  if (os.platform() === 'darwin' && commandExists.sync('sandbox-exec')) {
    return 'sandbox-exec';
  } else if (commandExists.sync('docker') && sandbox === true) {
    return 'docker';
  } else if (commandExists.sync('podman') && sandbox === true) {
    return 'podman';
  }
  
  return '';
}
```

### 3.5 Shell 命令执行

**文件路径**: `packages/core/src/tools/shell.ts`

```typescript
// 第 148-153 行: execute 方法
async execute(
  signal: AbortSignal,
  updateOutput?: (output: string | AnsiOutput) => void,
  shellExecutionConfig?: ShellExecutionConfig,
  setPidCallback?: (pid: number) => void,
): Promise<ToolResult> {
```

**文件路径**: `packages/core/src/services/shellExecutionService.ts`

```typescript
// 第 212-245 行: 核心执行逻辑
static async execute(
  commandToExecute: string,
  cwd: string,
  onOutputEvent: (event: ShellOutputEvent) => void,
  abortSignal: AbortSignal,
  shouldUseNodePty: boolean,
  shellExecutionConfig: ShellExecutionConfig,
): Promise<ShellExecutionHandle> {
  if (shouldUseNodePty) {
    // 尝试使用 node-pty (更好的终端模拟)
    const ptyInfo = await getPty();
    if (ptyInfo) {
      return await this.executeWithPty(...);
    }
  }
  // 回退到 child_process
  return this.childProcessFallback(...);
}
```

---

## 四、卷挂载详解

### 4.1 挂载表

| 主机路径 | 容器路径 | 权限 | 用途 | 代码位置 |
|----------|----------|------|------|----------|
| `$(pwd)` (当前工作目录) | 相同路径 | 读写 | 项目文件 | sandbox.ts:287 |
| `~/.gemini` | `/home/node/.gemini` | 读写 | 用户配置、会话 | sandbox.ts:303-306 |
| `/tmp` | `/tmp` | 读写 | 临时文件 | sandbox.ts:315 |
| `~` (homedir) | 相同路径 | 读写 | 主目录 | sandbox.ts:318-323 |
| `~/.config/gcloud` | 相同路径 | **只读** | GCloud 认证 | sandbox.ts:327-331 |
| `$GOOGLE_APPLICATION_CREDENTIALS` | 相同路径 | **只读** | ADC 文件 | sandbox.ts:335-344 |
| `$SANDBOX_MOUNTS` 指定路径 | 自定义 | 默认只读 | 额外挂载 | sandbox.ts:347-371 |
| `.gemini/sandbox.venv` | `$VIRTUAL_ENV` | 读写 | Python venv | sandbox.ts:537-554 |

### 4.2 环境变量传递

| 环境变量 | 用途 | 代码位置 |
|----------|------|----------|
| `SANDBOX` | 标识当前在容器内 | sandbox.ts:584 |
| `GEMINI_API_KEY` | API 密钥 | sandbox.ts:456-458 |
| `GOOGLE_API_KEY` | Google API 密钥 | sandbox.ts:459-461 |
| `GOOGLE_CLOUD_PROJECT` | GCP 项目 | sandbox.ts:494-499 |
| `TERM`, `COLORTERM` | 终端设置 | sandbox.ts:515-520 |
| `HOME` | 主目录 (UID 映射时) | sandbox.ts:636 |
| `HTTPS_PROXY`, `HTTP_PROXY` | 代理设置 | sandbox.ts:388-405 |

---

## 五、Docker 镜像分析

**文件路径**: `Dockerfile`

```dockerfile
FROM docker.io/library/node:20-slim

# 设置沙箱标识
ARG SANDBOX_NAME="gemini-cli-sandbox"
ENV SANDBOX="$SANDBOX_NAME"

# 安装开发工具
RUN apt-get update && apt-get install -y --no-install-recommends \
  python3 make g++ man-db curl dnsutils less jq bc \
  gh git unzip rsync ripgrep procps psmisc lsof socat \
  ca-certificates

# 配置 npm 全局目录
RUN mkdir -p /usr/local/share/npm-global \
  && chown -R node:node /usr/local/share/npm-global
ENV NPM_CONFIG_PREFIX=/usr/local/share/npm-global
ENV PATH=$PATH:/usr/local/share/npm-global/bin

# 切换到非 root 用户
USER node

# 安装 gemini-cli
COPY packages/cli/dist/google-gemini-cli-*.tgz /tmp/gemini-cli.tgz
COPY packages/core/dist/google-gemini-cli-core-*.tgz /tmp/gemini-core.tgz
RUN npm install -g /tmp/gemini-cli.tgz /tmp/gemini-core.tgz

CMD ["gemini"]
```

---

## 六、执行流程时序图

```
时间 ──────────────────────────────────────────────────────────────────▶

主机                                     容器
  │                                        
  │ 1. gemini -s -p "query"                
  │────┐                                   
  │    │ 解析参数                           
  │◀───┘                                   
  │                                        
  │ 2. loadSandboxConfig()                 
  │────┐                                   
  │    │ 检测到 sandbox=docker             
  │◀───┘                                   
  │                                        
  │ 3. refreshAuth()                       
  │────┐                                   
  │    │ OAuth 认证 (如需要)               
  │◀───┘                                   
  │                                        
  │ 4. start_sandbox()                     
  │────────────────────────────────────────┐
  │                                        │
  │     docker run -i --rm --init \        │
  │       --volume ... --env ... \         │
  │       gemini-cli-sandbox               │
  │                                        ▼
  │                                  ┌───────────┐
  │                                  │ 容器启动   │
  │                                  └─────┬─────┘
  │                                        │
  │                                  检测 SANDBOX 环境变量
  │                                        │
  │                                  跳过沙箱启动逻辑
  │                                        │
  │                                  执行实际 CLI 工作
  │                                        │
  │                                  ┌─────┴─────┐
  │                                  │           │
  │                             与 Gemini    执行 Shell
  │                             API 通信      命令
  │                                  │           │
  │                                  └─────┬─────┘
  │                                        │
  │         stdio 输出                      │
  │◀───────────────────────────────────────┤
  │                                        │
  │         退出码                          │
  │◀───────────────────────────────────────┤
  │                                        │
  │ 5. process.exit()                      │
  ▼                                        ▼
```

---

## 七、关键设计决策

### 7.1 为什么认证在主机完成？

OAuth 认证需要打开浏览器进行交互，容器内无法直接访问主机的桌面环境。因此认证必须在进入沙箱前完成。

**代码位置**: `gemini.tsx` 第 393-434 行

```typescript
// 在进入沙箱前刷新认证
if (!settings.merged.security.auth.useExternal) {
  try {
    await partialConfig.refreshAuth(settings.merged.security.auth.selectedType);
  } catch (err) {
    // 处理认证错误
  }
}
```

### 7.2 为什么需要 UID/GID 映射？

Linux 上，容器内的用户 ID 与主机不同会导致文件权限问题。Gemini CLI 在 Debian/Ubuntu 系统上自动创建与主机相同 UID/GID 的用户。

**代码位置**: `sandboxUtils.ts` 第 38-72 行

### 7.3 为什么使用 `--init` 参数？

`--init` 参数使容器使用 `tini` 作为 PID 1，正确处理信号转发和僵尸进程回收。

**代码位置**: `sandbox.ts` 第 268 行

```typescript
const args = ['run', '-i', '--rm', '--init', '--workdir', containerWorkdir];
```

---

## 八、自定义配置方法

### 8.1 项目级 Dockerfile

在 `.gemini/sandbox.Dockerfile` 放置自定义 Dockerfile，会自动用于构建项目专用沙箱镜像。

### 8.2 项目级启动脚本

在 `.gemini/sandbox.bashrc` 放置自定义脚本，容器启动时会自动 source。

### 8.3 额外挂载

使用 `SANDBOX_MOUNTS` 环境变量指定额外挂载：

```bash
export SANDBOX_MOUNTS="/data/models:/models:ro,/home/user/.ssh:/root/.ssh:ro"
```

### 8.4 端口转发

使用 `SANDBOX_PORTS` 环境变量指定需要转发的端口：

```bash
export SANDBOX_PORTS="3000,8080,5432"
```

---

## 九、调试方法

### 9.1 启用调试模式

```bash
DEBUG=1 gemini -s -p "命令"
```

### 9.2 检查容器内环境

```bash
gemini -s -p "run shell command: env | grep SANDBOX"
gemini -s -p "run shell command: mount | grep workspace"
```

### 9.3 查看 Docker 命令

设置 `DEBUG=1` 后，会输出完整的 `docker run` 命令。

---

## 十、总结

Gemini CLI 的沙箱设计采用"启动器-工作者"模式：

1. **主机 CLI** 作为启动器，负责认证和配置，然后启动容器
2. **容器内 CLI** 作为工作者，执行所有实际任务
3. 通过 **卷挂载** 实现文件双向同步
4. 通过 **环境变量** 传递配置和凭证
5. 通过 **stdio 继承** 实现输入输出透传

这种设计既保证了安全隔离，又保持了良好的用户体验。

---

*报告生成时间: 2026-02-05*
*分析版本: Gemini CLI (基于源码分析)*
