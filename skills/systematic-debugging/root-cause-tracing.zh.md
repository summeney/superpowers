# 根因追溯

## 概述

Bug 经常在调用栈深处显现（git init 在错误目录、文件创建在错误位置、数据库用错误路径打开）。你的本能是修复错误出现的地方，但那只是治标不治本。

**核心原则：** 沿着调用链反向追踪，直到找到原始触发点，然后在源头修复。

## 使用场景

```dot
digraph when_to_use {
    "Bug 在栈深处显现?" [shape=diamond];
    "能反向追溯吗?" [shape=diamond];
    "在症状处修复" [shape=box];
    "追溯到原始触发点" [shape=box];
    "更好：同时增加纵深防御" [shape=box];

    "Bug 在栈深处显现?" -> "能反向追溯吗?" [label="是"];
    "能反向追溯吗?" -> "追溯到原始触发点" [label="是"];
    "能反向追溯吗?" -> "在症状处修复" [label="否 - 死胡同"];
    "追溯到原始触发点" -> "更好：同时增加纵深防御";
}
```

**在以下情况使用：**
- 错误发生在执行深处（不在入口点）
- 堆栈跟踪显示很长的调用链
- 不清楚无效数据来自哪里
- 需要找出哪个测试/代码触发了问题

## 追溯过程

### 1. 观察症状
```
Error: git init failed in ~/project/packages/core
```

### 2. 找到直接原因
**什么代码直接导致了这个？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 问：是 谁调用了这个？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → 被 Session.initializeWorkspace() 调用
  → 被 Session.create() 调用
  → 被测试 at Project.create() 调用
```

### 4. 继续向上追溯
**传了什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 会解析为 `process.cwd()`
- 那就是源代码目录！

### 5. 找到原始触发点
**空字符串从哪里来的？**
```typescript
const context = setupCoreTest(); // 返回 { tempDir: '' }
Project.create('name', context.tempDir); // 在 beforeEach 之前访问！
```

## 添加堆栈跟踪

当你无法手动追溯时，添加工具：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：** 在测试中使用 `console.error()`（不是 logger - 可能会被抑制）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 寻找测试文件名
- 找出触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 找出是哪个测试导致污染

如果某东西在测试期间出现，但你不知道是哪个测试：

使用本目录中的二分脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐一运行测试，在第一个污染者处停下。详见脚本使用说明。

## 真实案例：空的 projectDir

**症状：** `.git` 创建在 `packages/core/`（源代码）中

**追溯链：**
1. `git init` 在 `process.cwd()` 运行 ← cwd 参数为空
2. WorktreeManager 用空 projectDir 调用
3. Session.create() 传递了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 最初返回 `{ tempDir: '' }`

**根本原因：** 顶级变量初始化访问了空值

**修复：** 将 tempDir 改为 getter，在 beforeEach 之前访问时抛出异常

**同时增加了纵深防御：**
- 第 1 层：Project.create() 验证目录
- 第 2 层：WorkspaceManager 验证非空
- 第 3 层：NODE_ENV 守卫拒绝在 tmpdir 外运行 git init
- 第 4 层：git init 前的堆栈跟踪日志

## 核心原则

```dot
digraph principle {
    "找到直接原因" [shape=ellipse];
    "能再追溯一层吗?" [shape=diamond];
    "反向追溯" [shape=box];
    "这是源头吗?" [shape=diamond];
    "在源头修复" [shape=box];
    "在每层添加验证" [shape=box];
    "Bug 不可能发生" [shape=doublecircle];
    "永远不要只修复症状" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "找到直接原因" -> "能再追溯一层吗?";
    "能再追溯一层吗?" -> "反向追溯" [label="是"];
    "能再追溯一层吗?" -> "永远不要只修复症状" [label="否"];
    "反向追溯" -> "这是源头吗?";
    "这是源头吗?" -> "反向追溯" [label="否 - 继续"];
    "这是源头吗?" -> "在源头修复" [label="是"];
    "在源头修复" -> "在每层添加验证";
    "在每层添加验证" -> "Bug 不可能发生";
}
```

**永远不要只在错误出现的地方修复。** 反向追溯找到原始触发点。

## 堆栈跟踪技巧

**在测试中：** 使用 `console.error()` 而不是 logger - logger 可能会被抑制
**操作之前：** 在危险操作之前记录日志，而不是失败之后
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整调用链

## 实际效果

来自调试会话（2025-10-03）：
- 通过 5 层追溯找到根本原因
- 在源头修复（getter 验证）
- 增加了 4 层防御
- 1847 个测试通过，零污染