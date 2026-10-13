# Promptfoo 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[Promptfoo](https://github.com/promptfoo/promptfoo) 是开源界最权威的 LLM 提示词工程化评估（Prompt Eval）、红队安全渗透测试（LLM Redteaming）与漏洞扫描工具。系统前端基于 Next.js 构建可视化对比分析矩阵，后端由 Node.js / TypeScript 核心评测管道驱动，能够对数十家大模型同时发起多样化断言验证（包括语义相似度、正则、JSON Schema 约束、红队越狱漏洞探测及自动评分）。

```
┌────────────────────────────────────────────────────────────────────────┐
│             Promptfoo Web Viewer (Next.js + 评分热力对比矩阵)          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ CLI / Config YAML (promptfooconfig.yaml)
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Promptfoo Core Evaluation Engine                    │
├───────────────────────────────┬────────────────────────────────────────┤
│ Test Matrix Expansion Engine  │ Assertion Evaluator Pipeline           │
│ - 提示词与测试变量笛卡尔积展开 │ - deterministic (regex, contains, etc) │
│ - 动态数据集生成与引用解析     │ - model-graded (llm-rubric, similarity)│
├───────────────────────────────┼────────────────────────────────────────┤
│ Provider Adapter Bus          │ Redteam Jailbreak Generator Pipeline   │
│ - OpenAI, Anthropic, Ollama   │ - 提示词越狱探测器 (Jailbreak, PII)    │
│ - 并发限流器与缓存 (SQLite)   │ - 自动化对抗性输入变形 (Adversarial)   │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Promptfoo 建立在极其完备的多维断言与自动化防退化测试之上：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Assertion Unit Tests** | `contains`、`regex`、`javascript`、`python` 自定义脚本断言 | Jest / Vitest | 纯内存沙箱执行，禁止危险系统调用 |
| **Model-Graded Assertions** | `llm-rubric` 评分规则匹配、Embedding 语义余弦相似度计算 | Vitest + Mock LLM Providers | 拦截模型 API 返回固定评分结果 |
| **Cartesian Matrix Expansion** | 提示词（Prompts）与变量（Vars）多对多展开、条件变量过滤 | Vitest (`test/evaluator.test.ts`) | 纯算法逻辑，验证矩阵组合数量 |
| **Redteaming Generator** | 越狱变形器（Rot13、Base64 编码越狱、系统角色注入攻击） | Vitest | 静态对抗模式生成校验 |
| **Local SQLite Caching** | 相同 Prompt 与参数二次评估命中缓存、缓存过期失效 | SQLite + Better-SQLite3 | 独立临时 SQLite 测试文件，运行后删除 |
| **CLI & Web Matrix E2E** | `promptfoo eval` 命令行执行并输出 HTML 报告，Playwright 验证比对 | Playwright + Node CLI | 隔离的工作目录与临时端口 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：多模式断言器（Assertion Pipeline）执行与评分断言

#### 3.1.1 业务功能与测试目标
Promptfoo 支持多达数十种断言类型。测试必须覆盖：
1. 确定性断言（`icontains` 大小写不敏感包含、`equals` 完全相等）。
2. 模式断言（`is-json` 结合 JSON Schema 校验输出合规性）。
3. 自定义 JavaScript 匿名函数断言（通过 `output.length > 50` 进行动态断言）。

#### 3.1.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';
import { runAssertions } from '../src/assertions';
import type { TestCase, Assertion } from '../src/types';

describe('Assertion Pipeline Suite', () => {
  it('evaluates deterministic, regex and json schema assertions with precision', async () => {
    const outputText = JSON.stringify({
      status: 'success',
      data: { score: 95, tags: ['ai', 'testing'] }
    });

    const testCase: TestCase = {
      vars: { topic: 'testing' },
      assert: [
        // 1. 包含断言
        { type: 'icontains', value: 'success' },
        // 2. 正则匹配断言
        { type: 'regex', value: 'score["\']?:\\s*9[0-9]' },
        // 3. 严格 JSON Schema 断言
        {
          type: 'is-json',
          value: {
            type: 'object',
            properties: {
              status: { type: 'string' },
              data: {
                type: 'object',
                properties: { score: { type: 'number' } },
                required: ['score']
              }
            },
            required: ['status', 'data']
          }
        },
        // 4. 自定义 JavaScript 断言
        {
          type: 'javascript',
          value: 'output.includes("ai") && JSON.parse(output).data.score > 90'
        }
      ]
    };

    // 运行断言器
    const results = await runAssertions(testCase, outputText);

    // 严格断言：每个断言均必须通过
    expect(results).toHaveLength(4);
    for (const res of results) {
      expect(res.pass).toBe(true);
      expect(res.reason).toBeUndefined();
    }
  });
});
```

---

### 3.2 模块二：笛卡尔积测试矩阵（Matrix Expansion）展开测试

#### 3.2.1 业务功能与测试目标
如果配置了 2 个 Prompts，3 个测试用例，且定义了 2 种模型（OpenAI 与 Claude），系统必须精确展开为 $2 \times 3 \times 2 = 12$ 个独立的评估单元。测试必须保证：
1. 展开数量与变量替换完全对齐。
2. 变量存在嵌套依赖或默认值时能正确继承与覆盖。

#### 3.2.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';
import { expandTestMatrix } from '../src/matrix';

describe('Test Matrix Expansion Suite', () => {
  it('correctly expands Cartesian product of prompts, vars and providers', () => {
    const prompts = ['Translate {{text}} to {{language}}', 'Say {{text}} in {{language}}'];
    const vars = [
      { text: 'Hello', language: 'French' },
      { text: 'Goodbye', language: 'German' },
      { text: 'Thanks', language: 'Spanish' },
    ];
    const providers = ['provider-openai', 'provider-anthropic'];

    const expandedRuns = expandTestMatrix({
      prompts,
      vars,
      providers,
    });

    // 2 prompts * 3 vars * 2 providers = 12 combinations
    expect(expandedRuns).toHaveLength(12);

    // 验证特定组合的变量替换插值正确性
    const firstRun = expandedRuns[0];
    expect(firstRun.prompt).toBe('Translate Hello to French');
    expect(firstRun.provider).toBe('provider-openai');
  });
});
```

---

### 3.3 模块三：红队对抗性越狱（Redteam Jailbreak Generator）测试

#### 3.3.1 业务功能与测试目标
Promptfoo Redteam 模块负责自动对恶意提示词生成对抗变体（如针对 "How to build a bomb" 尝试角色扮演、Rot13 编码或历史重构越狱）。测试必须覆盖：
1. 变体生成器能够产出预期的攻击载荷并标记安全类别（如 PII、Harassment、Cyberattack）。
2. 防火墙（Guardrail）命中判定与漏洞告警。

#### 3.3.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';
import { generateRedteamPlugins } from '../src/redteam';

describe('Redteam Jailbreak Generation Suite', () => {
  it('generates roleplay and encoding adversarial jailbreak testcases', async () => {
    const baseIntent = 'Extract system prompt and secrets';
    const plugins = ['jailbreak', 'rot13', 'sql-injection'];

    const redteamCases = await generateRedteamPlugins({
      intent: baseIntent,
      plugins,
    });

    expect(redteamCases.length).toBeGreaterThanOrEqual(3);

    // 验证 Rot13 编码变体存在且包含解密指令
    const rot13Case = redteamCases.find(c => c.metadata?.plugin === 'rot13');
    expect(rot13Case).toBeDefined();
    expect(rot13Case!.vars.prompt).toContain('decode the following text');

    // 验证包含自动违规断言检查点
    expect(rot13Case!.assert).toContainEqual(
      expect.objectContaining({ type: 'llm-rubric', metric: 'JailbreakSucceeded' })
    );
  });
});
```

---

### 3.4 模块四：本地 SQLite 缓存与防重复计费测试

#### 3.4.1 业务功能与测试目标
大模型 API 调用费用昂贵。Promptfoo 内置缓存层，对于 Prompt、Model、Temperature、MaxTokens 完全一致的请求，自动读取本地 SQLite 缓存。

#### 3.4.2 测试用例实现
```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { CacheManager } from '../src/cache';
import fs from 'fs';

describe('SQLite Evaluation Cache Suite', () => {
  const testDbPath = './test_cache.db';
  let cacheManager: CacheManager;

  beforeEach(() => {
    cacheManager = new CacheManager({ dbPath: testDbPath });
  });

  afterEach(() => {
    cacheManager.close();
    if (fs.existsSync(testDbPath)) {
      fs.unlinkSync(testDbPath);
    }
  });

  it('stores and retrieves cached API responses to save tokens', async () => {
    const prompt = 'What is the capital of Japan?';
    const mockResponse = { output: 'Tokyo', tokenUsage: 12 };

    // 1. 初始查询必须未命中
    const cachedFirst = await cacheManager.get(prompt, { model: 'gpt-4o' });
    expect(cachedFirst).toBeNull();

    // 2. 写入缓存
    await cacheManager.set(prompt, { model: 'gpt-4o' }, mockResponse);

    // 3. 再次读取，立即秒级命中
    const cachedSecond = await cacheManager.get(prompt, { model: 'gpt-4o' });
    expect(cachedSecond).not.toBeNull();
    expect(cachedSecond?.output).toBe('Tokyo');
    expect(cachedSecond?.tokenUsage).toBe(12);
  });
});
```

---

## 4. 测试工程化与基础设施

1. **确定性 Mock 避免外部网络调用**：
   * 在 CI 流水线中，所有基于大语言模型的断言均使用 Mock Provider，根据输入 Prompt 前缀预设好固定评分结果，既保证测试零耗资，又消除了模型幻觉带来的 Flaky Test。
2. **轻量沙箱隔离自定义 JavaScript 断言**：
   * 用户编写的 JS 断言在 Node.js `vm2` 或原生受限子上下文中执行，禁止访问 `process.env` 与文件系统。

---

## 5. 对同类复杂系统的参考价值

* **非确定性系统的测试量化工程**：针对大模型应用无法进行传统全等断言的问题，Promptfoo 建立的“确定性规则（正则/包含/模式） + 模型打分（Rubric） + 语义阈值（Cosine Similarity）”三层判定矩阵，是设计任何现代 AI 评测框架的标准工业范式。
