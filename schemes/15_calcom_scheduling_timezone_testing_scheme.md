# Cal.com 自动化测试方案与用例设计深度剖析

## 1. 项目背景与系统架构特征

### 1.1 系统定位与技术栈
[Cal.com](https://github.com/calcom/cal.com) 是全球领先的企业级开源会议预约与日程调度 Web 平台（Calendly 开源替代品）。系统基于全栈 TypeScript 构建，采用 Next.js (App Router)、React、tRPC（端到端强类型 RPC）、Prisma ORM 以及 PostgreSQL。

### 1.2 被测系统核心组成
1. **可用性与时间槽位引擎（Availability & Slot Computation）**：根据发起人配置的工作时间段、已有日历忙闲状态、跨时区转换及缓冲时间（Buffer time）动态计算空闲预约槽位。
2. **多源第三方日历同步（Calendar Integrations）**：双向实时同步 Google Calendar、Outlook、Apple Calendar 等日历事件。
3. **高并发抢占与防双重预订（Double Booking Prevention & Concurrency Control）**：保证在极端并发场景下，同一个预约时间段绝对不会被两个用户同时预订。
4. **自定义表单、支付与动态路由（Dynamic Booking Form & Stripe Billing）**：收集参会人问卷、收取预约押金、动态根据回答路由到不同主持人。

### 1.3 自动化测试的主要难点与挑战
* **全球夏令时/冬令时（DST）与跨时区转换的边缘陷阱**：预约发起人与参会人处于不同大洲，且处于夏令时交替日（如每年 3 月和 11 月）时的时间计算。
* **高并发预约抢占竞态（Race Condition）**：毫秒内两名用户在网页端同时点击同一个 10:00-10:30 槽位。
* **复杂的第三方 OAuth 与日历 API 依赖**：如何在离线测试中可靠模拟 Google/Outlook 日历繁忙事件的双向拉取与推送。

---

## 2. 测试架构全景与分层策略

Cal.com 建立了类型安全（Type-Safe）与行为驱动相结合的端到端测试防线：

```
┌────────────────────────────────────────────────────────┐
│             Playwright End-to-End User Journeys        │  端到端交互层：驱动日历组件选择槽位、填写表单并提交
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│      tRPC Router & Middleware Integration Tests        │  业务接口层：端到端类型校验、鉴权拦截与错误码断言
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│         Slots & Timezone Calculation Core Tests        │  时区计算层：基于纯数学与日期库穷尽跨时区槽位矩阵
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┘
│         Isolated PostgreSQL Prisma Test Database       │  数据持久层：利用事务回滚保持每个测试用例状态纯净
└────────────────────────────────────────────────────────┘
```

---

## 3. 核心功能及测试用例设计剖析

### 3.1 功能一：跨时区与夏令时交替（DST）时间槽位精准计算

#### 3.1.1 业务功能背景
发起人在“亚洲/东京”（UTC+9）设置工作时间为周一 09:00 - 17:00；参会人在“美洲/纽约”（UTC-5，夏令时期间为 UTC-4）打开预约网页。系统必须在网页日历上精准呈现换算后的本地时间槽位，且在夏令时切换当天时间不能发生偏差 1 小时的 Bug。

#### 3.1.2 测试用例设计方案
* **测试用例实现（Vitest 纯逻辑算法测试）**：
```typescript
import { describe, it, expect } from 'vitest';
import { getSchedule } from '@calcom/features/schedules/lib/getSchedule';

describe('Timezone & DST Slot Computation', () => {
  it('correctly shifts slots across DST transition date for user in New York booking Tokyo host', async () => {
    // 1. 发起人配置：时区为 Asia/Tokyo，周一上午 9:00 - 12:00
    const hostSchedule = {
      timeZone: 'Asia/Tokyo',
      availability: [{ days: [1], startTime: '09:00', endTime: '12:00' }],
    };

    // 2. 模拟参会人在夏令时切换日（2026-03-08）查看可用槽位
    const attendeeTimeZone = 'America/New_York';
    const slots = await getSchedule({
      dateFrom: '2026-03-08',
      dateTo: '2026-03-09',
      hostSchedule,
      timeZone: attendeeTimeZone,
    });

    // 3. 东京 09:00 对应纽约前一天晚上 19:00 或 20:00（依据夏令时偏移精确计算）
    expect(slots.length).toBeGreaterThan(0);
    expect(slots[0].time).toBe('2026-03-08T19:00:00.000-05:00');

    // 4. 断言时间槽位时长严格为 30 分钟步长，无时间重叠
    const firstSlotDuration = new Date(slots[0].endTime).getTime() - new Date(slots[0].time).getTime();
    expect(firstSlotDuration).toBe(30 * 60 * 1000);
  });
});
```

---

### 3.2 功能二：并发防双重预订（Double Booking Prevention）

#### 3.2.1 业务功能背景
当两个参会人在同一秒内点击并提交同一个时间段（例如周二 14:00）的预约请求时，系统底层必须依赖数据库行级锁或乐观锁，确保一人预订成功，另一人立即收到已被占用的提示并引导重新选时间。

#### 3.2.2 测试用例设计方案
* **测试用例实现（tRPC 路由并发压力集成测试）**：
```typescript
import { describe, it, expect } from 'vitest';
import { createCaller } from '@calcom/trpc/server/routers/_app';

describe('Concurrency Control: Double Booking Guard', () => {
  it('ensures only one booking succeeds when two requests hit the same slot concurrently', async () => {
    const callerAlice = createCaller({ user: { id: 'user-alice' } });
    const callerBob = createCaller({ user: { id: 'user-bob' } });

    const targetSlot = '2026-04-15T10:00:00.000Z';
    const bookingPayload = {
      eventTypeId: 1,
      start: targetSlot,
      end: '2026-04-15T10:30:00.000Z',
    };

    // 并发同时发起两笔预订请求（Promise.allSettled 模拟竞争）
    const [resultAlice, resultBob] = await Promise.allSettled([
      callerAlice.bookings.create(bookingPayload),
      callerBob.bookings.create(bookingPayload),
    ]);

    // 统计成功与失败数量
    const fulfilled = [resultAlice, resultBob].filter(r => r.status === 'fulfilled');
    const rejected = [resultAlice, resultBob].filter(r => r.status === 'rejected');

    // 严格断言：必须且仅有 1 笔成功，另 1 笔被安全拒绝并抛出已占用异常
    expect(fulfilled).toHaveLength(1);
    expect(rejected).toHaveLength(1);
    
    if (rejected[0].status === 'rejected') {
      expect(rejected[0].reason.message).toContain('booking_conflict_time_already_taken');
    }
  });
});
```

---

### 3.3 功能三：Playwright 完整预约链路端到端断言

#### 3.3.1 业务功能背景
用户访问公共个人预约页 `https://cal.com/john`，点击日期、点击时间槽、填写姓名邮箱并点击提交，页面应跳转至预约成功页面，并生成 `.ics` 日历附件下载链接。

#### 3.3.2 测试用例设计方案
* **测试用例实现**：
```typescript
test('complete booking flow via web UI', async ({ page }) => {
  await page.goto('/john/30min');

  // 1. 点击日历网格中第一个可用的日期
  const availableDay = page.locator('[data-testid="day"][data-disabled="false"]').first();
  await availableDay.click();

  // 2. 选择可用的具体时间槽并进入表单
  const slotButton = page.locator('[data-testid="time"]').first();
  await slotButton.click();

  // 3. 填写参会人信息表单
  await page.fill('input[name="name"]', 'Bob Smith');
  await page.fill('input[name="email"]', 'bob@example.com');
  await page.click('button[type="submit"]');

  // 4. 断言成功确认页面与事件详情
  await expect(page.locator('[data-testid="booking-success"]')).toBeVisible();
  await expect(page.locator('.booking-attendee-name')).toHaveText('Bob Smith');
});
```

---

## 4. 自动化测试基础设施与 CI 质量保障

1. **类型安全端到端链路（End-to-End Type Safety）**：
   * 前端直接引用后端 tRPC 路由的类型定义，在运行测试前由 TypeScript 编译器静态拦截所有入参不匹配的请求。
2. **Prisma 数据库事务回滚模式**：
   * 在单元/集成测试中，每个用例运行在独立的 Prisma Transaction 中，用例结束立即 Rollback，省去繁琐的数据清理脚本，执行速度极快。

---

## 5. 对复杂 Web 系统的实战借鉴意义

* **时区与时间敏感系统必须脱离本地机器时间**：测试用例中所有涉及日历、工单、排班的时间，必须显式声明包含时区偏移量的 ISO 字符串，严禁依赖执行机器（CI Runner）的本地时区。
* **高价值资产抢占必须编写真正的并发竞争测试**：对于预约、购票、抢单等存在“库存/时间槽唯一性”的系统，必须在集成测试中编写 `Promise.all` 并发压测，验证底层锁机制和错误码返回。
