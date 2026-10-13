# Vitest 自动化测试方案与隔离运行内核全集深度剖析

## 1. 项目背景与系统架构全景

[Vitest](https://github.com/vitest-dev/vitest) 是现代前端与 Node.js 生态中最快、最具扩展性的原生测试框架。它直接依托 **Vite** 的极速按需编译与模块图（Module Graph）转换能力，彻底解决了传统 Jest 在 ESM（ECMAScript Modules）与 TypeScript 项目中编译缓慢、配置繁杂的痛点。其核心特性包括：**Tinypool 多线程/进程工作池隔离、基于底层 Module Loader 的零污染 Mock 系统、快照序列化引擎以及原生浏览器真实渲染模式（Browser Mode）**。

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Vitest CLI / VS Code IDE                        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ WebSocket / RPC 双向通道
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Vitest Core Runner (Vite Pipeline)                  │
├───────────────────────────────┬────────────────────────────────────────┤
│ Module Graph & HMR Engine     │ Pool Scheduling Engine (Tinypool)      │
│ - 智能按需转换 (Vite Transform│ - threads (Worker Threads 极速执行)   │
│ - 依赖变更热测试重跑 (HMR)    │ - forks (独立进程隔离, 针对原生 C 插件)│
├───────────────────────────────┼────────────────────────────────────────┤
│ Module Mocking (vi.mock)      │ Snapshot Serialization Engine          │
│ - ESM 动态代理拦截与导出替换  │ - 格式化 AST 语法树比对               │
│ - 自动恢复与全局状态清理      │ - 脏快照智能过期标记与清理             │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Vitest 本身作为一个测试执行引擎，其内部的自我验证（Self-testing）用例矩阵涵盖极广：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Transform & Vite Unit** | TypeScript 转换、JSX 剥离、Decorators 支持、CSS/资源跳过 | Vitest 自我运行 | 内存模块字符串转换 |
| **Worker Isolation Tests** | Worker 线程崩溃捕获、全局变量（`globalThis`）跨用例零污染 | Tinypool 多线程池 | 每次用例重置 Module Context |
| **Module Mocking Tests** | `vi.mock`、`vi.spyOn`、`vi.hoisted` 静态提升与动态还原 | Vitest Mock Pipeline | Proxy 代理与 ESM Loader 拦截 |
| **Snapshot Engine Tests** | 内联快照写入源文件、外部 `.snap` 文件比对、属性匹配器 | Vitest Snapshot Suite | 临时测试文件与 AST 注入比对 |
| **Browser Mode Integration** | 在真实的无头 Chromium / WebKit 浏览器窗口中运行断言 | Playwright + Vite DevServer | 隔离的 Browser Context 实例 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：Tinypool 工作线程池隔离与全局变量（Global Scope）零污染测试

#### 3.1.1 业务功能与测试目标
Vitest 默认在多 Worker 线程中并行运行测试文件。测试必须保证：
1. 测试文件 A 污染了全局变量（如 `(globalThis as any).foo = 'bar'`），不能对测试文件 B 造成任何可见性泄露。
2. 当某个 Worker 发生内存溢出或调用 `process.exit(1)` 时，主调度器能精准捕获错误并将其余测试派发至健康的 Worker，不发生级联死锁。

#### 3.1.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';

describe('Worker Pool Global Isolation Suite', () => {
  it('ensures global scope pollution is strictly contained within worker lifecycle', async () => {
    // 1. 检查初始状态无污染
    expect((globalThis as any).__vitest_leak_check__).toBeUndefined();

    // 2. 注入全局变量
    (globalThis as any).__vitest_leak_check__ = 'dirty_value';
    expect((globalThis as any).__vitest_leak_check__).toBe('dirty_value');

    // 3. 在下一个独立测试块或 Worker 执行中，断言隔离层自动清洗了此状态
    // Vitest 的 isolate: true 保证了全局对象的按需重置
  });

  it('safely catches native unhandled rejection without terminating worker process pool', async () => {
    let unhandledTriggered = false;
    process.once('unhandledRejection', () => {
      unhandledTriggered = true;
    });

    // 触发一个未捕获的 Promise 异常
    Promise.reject(new Error('Simulated unhandled failure'));

    // 等待一个事件周期
    await new Promise(resolve => setTimeout(resolve, 50));
    expect(unhandledTriggered).toBe(true);
  });
});
```

---

### 3.2 模块二：静态提升与动态模块打桩（`vi.hoisted` 与 `vi.mock`）底层拦截测试

#### 3.2.1 业务功能与测试目标
由于 ESM 规范中 `import` 语句在代码执行前进行静态提升绑定，常规 Mock 手段无法在模块导入前拦截其依赖。Vitest 通过 Vite 转换插件在 AST 阶段将 `vi.mock` 和 `vi.hoisted` 强制提升到文件最顶端。测试必须验证：
1. 模块导入前完成依赖打桩。
2. 调用 `vi.restoreAllMocks()` 或 `vi.resetModules()` 后，物理导入真实原始实现。

#### 3.2.2 测试用例实现
```typescript
import { describe, it, expect, vi } from 'vitest';

// 1. 静态提升测试块：保证在任何 import 发生前执行
const { mockedFetch } = vi.hoisted(() => {
  return { mockedFetch: vi.fn().mockResolvedValue({ status: 'mocked' }) };
});

// 2. 模块级打桩
vi.mock('../src/api-client', () => {
  return {
    fetchUserData: mockedFetch
  };
});

import { getUserProfile } from '../src/user-service';

describe('Module Mocking & Hoisting Engine Suite', () => {
  it('invokes hoisted mock instead of actual production network module', async () => {
    const profile = await getUserProfile(101);

    // 验证调用了打桩实现
    expect(mockedFetch).toHaveBeenCalledWith(101);
    expect(profile).toEqual({ status: 'mocked' });

    // 3. 动态恢复测试
    vi.restoreAllMocks();
    expect(mockedFetch.mock.calls.length).toBe(1);
  });
});
```

---

### 3.3 模块三：快照序列化与属性非对称匹配器（Asymmetric Matchers）测试

#### 3.3.1 业务功能与测试目标
快照测试支持对时间戳、自增 ID 等动态字段使用非对称匹配器（如 `expect.any(Date)`），而对其余静态属性进行严格的快照比对。测试必须验证：
1. 包含动态属性的对象能够稳定通过快照，不产生误报。
2. 静态属性一旦变更，差异以精确的统一 Diff（Unified Diff）格式展示。

#### 3.3.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';

describe('Snapshot Asymmetric Matching Suite', () => {
  it('matches complex object snapshot while tolerating dynamic timestamp and id', () => {
    const userPayload = {
      id: Math.floor(Math.random() * 10000),
      createdAt: new Date(),
      username: 'developer_01',
      role: 'ADMIN',
      settings: {
        theme: 'dark',
        notifications: true
      }
    };

    // 强断言：结合非对称匹配器进行快照比对
    expect(userPayload).toMatchInlineSnapshot(
      {
        id: expect.any(Number),
        createdAt: expect.any(Date),
      },
      `
      {
        "createdAt": Any<Date>,
        "id": Any<Number>,
        "role": "ADMIN",
        "settings": {
          "notifications": true,
          "theme": "dark",
        },
        "username": "developer_01",
      }
    `
    );
  });
});
```

---

### 3.4 模块四：浏览器模式（Browser Mode）原生 DOM 驱动测试

#### 3.4.1 业务功能与测试目标
不同于 JSDOM 模拟环境，Vitest 浏览器模式直接将用例运行在无头 Chromium 内部。测试必须验证：
1. 真实的 CSS 样式计算（`window.getComputedStyle`）。
2. WebGL Canvas 真实渲染与浏览器原生事件循环机制。

#### 3.4.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';

describe('Vitest Native Browser Mode Suite', () => {
  it('accesses genuine layout engine and computes real DOM bounding box', () => {
    // 创建一个绝对定位的 DOM 节点
    const div = document.createElement('div');
    div.style.width = '200px';
    div.style.height = '100px';
    div.style.backgroundColor = 'red';
    document.body.appendChild(div);

    // 1. 获取真实渲染盒模型（在 JSDOM 中此类计算通常返回 0）
    const rect = div.getBoundingClientRect();
    expect(rect.width).toBe(200);
    expect(rect.height).toBe(100);

    // 2. 校验真实计算样式
    const style = window.getComputedStyle(div);
    expect(style.backgroundColor).toBe('rgb(255, 0, 0)');

    document.body.removeChild(div);
  });
});
```

---

## 4. 测试工程化与基础设施

1. **Tinypool 线程池故障隔离**：
   * 在多核 CPU 机器上，Vitest 自动计算 `Math.max(os.cpus().length - 1, 1)` 并发 Worker，并通过基于 MessagePort 的传输机制消除序列化损耗。
2. **ESM 原生模块缓存清理**：
   * 为了解决 Node.js 原生 ESM 无法被清空的问题，Vitest 在内部 Module Runner 中为每次执行追加动态版本查询参数（`?v=timestamp`），实现真正的模块热重置。

---

## 5. 对同类复杂系统的参考价值

* **开发类基础设施的元测试设计**：编写测试运行器或编译器本身的自动化测试时，必须把“隔离”作为第一优先级。通过多 Worker 线程池 + 动态打桩拦截 + 虚拟快照的比对策略，是构建下一代极速开发工具链的最佳范本。
