# Vitest 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Vitest](https://github.com/vitest-dev/vitest) 是基于 Vite 构建的新一代极速单元测试运行器，原生支持 ESM、TypeScript、JSX，并提供开箱即用的多线程并发、快照测试、覆盖率统计与模拟（Mocking）能力。

### 1.2 被测系统核心组成
1. **测试调度与工作线程池（Worker Pool & Tinypool）**：负责跨 CPU 核心分发测试文件，提供基于 Worker Threads 或 Child Processes 的运行时环境隔离。
2. **转换引擎与模块解析（Vite Transform Pipeline）**：利用 Vite 管道实时将 TypeScript/JSX 编译为原生 JS，并保持 SourceMap 映射。
3. **快照测试引擎（Snapshot Engine）**：负责文件快照（`*.snap`）与内联快照（`toMatchInlineSnapshot`，直接重写源码 AST）的序列化与比对。
4. **监听模式与增量重跑（Watch Mode & HMR Graph）**：解析源码与测试文件的依赖反向引用图，文件改动时毫秒级重跑最小关联测试集。

### 1.3 自动化测试的主要难点与挑战
* **“用测试工具测试自身”的自举（Bootstrapping）悖论**：如何在测试执行器自身被修改或破损时，依然能够获得可信赖的测试信号。
* **跨线程全局状态污染（Global Leakage）**：Worker 线程重用时，如何测试不同测试文件之间的全局变量、定时器或未清理事件监听不发生穿透。
* **内联快照的 AST 回写破坏性**：测试用例直接读取并修改自身的 `.ts` 源文件代码，稍有不慎会导致源码文件损坏。

---

## 2. 测试架构全景与分层策略

Vitest 采用了专门的自举隔离集成测试架构：

```
┌────────────────────────────────────────────────────────┐
│             CLI Spawn & Subprocess E2E Tests           │  命令行黑盒层：通过 Node 子进程执行编译后的 vitest run
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│          Worker Pool Isolation Integration Tests       │  并发隔离层：验证 Worker 线程池内存与环境变量纯净度
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      Snapshot & AST Code Modification Tests            │  语法树操作层：针对内联快照源码自动改写的容错断言
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│       Core Matchers & Utils Unit Tests                 │  单元层：断言工具库与序列化纯函数逻辑验证
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：Worker 并发线程池环境隔离与防污染断言

#### 3.1.1 业务功能背景
当多个测试文件在同一个线程池中先后执行时，如何确保一个文件对 `globalThis` 的修改、环境变更改写不会污染后续用例？

#### 3.1.2 测试用例设计方案
* **测试用例编排**：设计一对专门用于相互探测的受测文件。
  * `fixtures/isolate-a.test.ts`：故意修改全局变量：
    ```typescript
    test('pollutes global', () => {
      // @ts-ignore
      globalThis.LEAKED_VAR = 'dirty_value';
    });
    ```
  * `fixtures/isolate-b.test.ts`：断言全局变量不存在：
    ```typescript
    test('expects clean global', () => {
      // @ts-ignore
      expect(globalThis.LEAKED_VAR).toBeUndefined();
    });
    ```
* **主测试断言实现**：
```typescript
test('workers should strictly isolate global environment between test files', async () => {
  // 通过 Vitest 内部 API 启动受测测试套件
  const { stdout, stderr, exitCode } = await runVitest({
    include: ['test/fixtures/isolate-*.test.ts'],
    pool: 'threads',
    poolOptions: { threads: { isolate: true } }
  });

  // 断言：两个测试必须均通过，证明隔离机制有效
  expect(exitCode).toBe(0);
  expect(stdout).toContain('2 passed');
});
```

---

### 3.2 功能二：内联快照（Inline Snapshot）AST 源码回写测试

#### 3.2.1 业务功能背景
Vitest 的 `expect(val).toMatchInlineSnapshot()` 能够自动将执行结果写回到测试文件源代码的参数括号中。这个功能涉及对 TypeScript 源码的词法分析与 AST 覆写。

#### 3.2.2 测试用例设计方案
* **测试步骤**：
  1. 在临时目录中动态生成一份原始 `.test.ts` 源码文件：
     ```typescript
     import { test, expect } from 'vitest';
     test('demo', () => {
       expect({ name: 'Vitest' }).toMatchInlineSnapshot();
     });
     ```
  2. 以 `-u`（更新快照）参数通过子进程运行 Vitest。
  3. 用例结束后，读取该临时测试文件的物理磁盘文本内容。
  4. **断言验证**：
     * 文本中必须被自动注入字符串格式化结果：`toMatchInlineSnapshot('"{ name: \\"Vitest\\" }"')`。
     * 源代码中的原始格式、缩进与未变更的代码行绝对未被损坏。

---

### 3.3 功能三：文件变更监听与增量依赖图分析（Watch Mode HMR）

#### 3.3.1 业务功能背景
在 Watch 模式下，当开发者修改 `src/utils.ts` 时，Vitest 只应运行直接或间接导入了 `utils.ts` 的测试文件，而不应重跑整个项目。

#### 3.3.2 测试用例设计方案
* **测试用例实现**：
```typescript
test('watch mode re-runs only impacted tests', async () => {
  // 1. 启动交互式 Watch 进程
  const vitestProcess = startVitestWatch(['test/fixtures/watch-project']);
  await vitestProcess.waitForInitialRun();

  // 2. 模拟修改被依赖的核心组件源文件
  await updateSourceFile('src/target.ts', '// trigger change');

  // 3. 监听控制台输出流中的测试统计
  const rerunReport = await vitestProcess.waitForNextRerun();
  
  // 4. 断言：仅执行了直接引用 target.ts 的测试用例
  expect(rerunReport.executedFiles).toEqual(['test/target.spec.ts']);
  expect(rerunReport.executedFiles).not.toContain('test/unrelated.spec.ts');
  
  await vitestProcess.kill();
});
```

---

### 3.4 功能四：SourceMap 错误堆栈精确行号映射断言

#### 3.4.1 业务功能背景
TypeScript 经过编译打包后代码行数会发生变化，若断言失败时报错指向编译后的中间文件，开发者将无法排错。

#### 3.4.2 测试用例设计方案
* **测试用例实现**：
  * 构建一份具有多行 JSX、装饰器与类型转换的 TypeScript 测试用例，故意在第 47 行触发 `expect(1).toBe(2)`。
  * 捕获进程输出的 Error Stack Trace。
  * 精确断言堆栈中输出的文件名与行号是 `my-case.test.tsx:47`，而非任何临时转译文件的行号。

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **跨平台换行符与路径标准化**：
   * 在快照比对与 AST 序列化中强制执行 `\r\n` 转 `\n` 规范化，确保在 Windows 与 Linux 环境下生成的快照哈希 100% 一致。
2. **多生态插件集成测试矩阵**：
   * 分别在真实的 React、Vue、Svelte、Solid 项目模板中运行 Vitest，验证各前端框架在 Node/JSDOM 环境下的渲染与挂载能力。

---

## 5. 对复杂系统自动化测试设计的实战借鉴意义

* **元工具（Meta-tooling）黑盒化验证**：对于构建工具、编译器、测试工具等元系统，必须通过起子进程黑盒执行打包后的二进制来构建最顶层的端到端测试防线。
* **隔离性与防污染测试是并发系统的核心**：在构建任何多任务并发调度系统时，必须专门编写“故意制造全局污染”的反向用例，确保隔离边界的坚韧度。
* **建立依赖图谱的精准测试思想**：大型系统在进行本地调试与持续集成时，可借鉴模块依赖反向引用图，实现“变更代码与测试用例的精准关联与最小集重跑”。
