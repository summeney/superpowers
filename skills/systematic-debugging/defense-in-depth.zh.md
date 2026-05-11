# 纵深防御验证

## 概述

当你修复一个由无效数据引起的 bug 时，在一个地方添加验证似乎就足够了。但这个单一检查可能被不同的代码路径、重构或 mocks 所绕过。

**核心原则：** 在数据经过的每一层都进行验证。从结构上让 bug 不可能发生。

## 为什么要多层防护

单层验证："我们修复了这个 bug"
多层防护："我们让这个 bug 不可能发生"

不同层捕获不同的情况：
- 入口验证捕获大多数 bug
- 业务逻辑捕获边界情况
- 环境防护阻止特定上下文中的危险操作
- 调试日志帮助其他层失效时进行排查

## 四层防护

### 第一层：入口点验证
**目的：** 在 API 边界处拒绝明显无效的输入

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... 继续执行
}
```

### 第二层：业务逻辑验证
**目的：** 确保数据对这个操作来说是合理的

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... 继续执行
}
```

### 第三层：环境防护
**目的：** 阻止在特定上下文中的危险操作

```typescript
async function gitInit(directory: string) {
  // 在测试中，拒绝在临时目录外执行 git init
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... 继续执行
}
```

### 第四层：调试插桩
**目的：** 捕获上下文以便进行排查

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... 继续执行
}
```

## 应用此模式

当你发现一个 bug 时：

1. **追踪数据流** - 坏值从哪里产生？在哪里被使用？
2. **映射所有检查点** - 列出数据经过的每个点
3. **在每一层添加验证** - 入口、业务、环境、调试
4. **测试每一层** - 尝试绕过第一层，验证第二层能捕获它

## 会话中的例子

Bug：空的 `projectDir` 导致在源代码目录中执行了 `git init`

**数据流：**
1. 测试设置 → 空字符串
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` 在 `process.cwd()` 中执行

**添加的四层防护：**
- 第一层：`Project.create()` 验证不为空/存在/可写
- 第二层：`WorkspaceManager` 验证 projectDir 不为空
- 第三层：`WorktreeManager` 在测试中拒绝在 tmpdir 外执行 git init
- 第四层：git init 前的堆栈跟踪日志

**结果：** 所有 1847 个测试通过，bug 不可能重现

## 关键洞察

四层都是必要的。在测试过程中，每一层都捕获了其他层遗漏的 bug：
- 不同的代码路径绕过了入口验证
- mocks 绕过了业务逻辑检查
- 不同平台上的边界情况需要环境防护
- 调试日志识别了结构性的误用

**不要在单个验证点停下。** 在每一层都添加检查。
