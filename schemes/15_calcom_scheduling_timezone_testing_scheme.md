# Cal.com 自动化测试方案与用例全集深度剖析

## 1. 项目背景与系统架构全景

[Cal.com](https://github.com/calcom/cal.com) 是全球领先的企业级日程预约调度基础设施平台。前端基于 Next.js + React + Tailwind CSS，后端依托 Next.js API Routes + Prisma ORM + PostgreSQL 构建高可靠性调度引擎。系统核心复杂度在于**多参与者跨时区空闲计算（Availability Calculation）、夏令时切换补偿、第三方日历双向同步（Google/Outlook）以及 Webhook 回调幂等控制**。

```
┌────────────────────────────────────────────────────────────────────────┐
│             Cal.com Booking Flow (Next.js App Router)                  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ TRPC / Server Actions
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Cal.com Core Engine (Node.js/Prisma)                │
├───────────────────────────────┬────────────────────────────────────────┤
│ Availability Schedule Engine  │ Timezone & DST Offset Resolver         │
│ - 用户工作时间区间求交集      │ - 协调世界时 (UTC) 与本地时区双向换算   │
│ - 已预约时间块占位剔除        │ - 跨夏令时/冬令时边界时间段补偿         │
├───────────────────────────────┼────────────────────────────────────────┤
│ Calendar Sync Adapter Bus     │ Webhook & Idempotency Pipeline         │
│ - Google / Outlook / Apple    │ - 签名计算与安全校验                   │
│ - 增量同步与网络异常重试      │ - 重复回调防抖与数据库唯一事务约束     │
└───────────────────────────────┴────────────────────────────────────────┘
```

---

## 2. 测试架构全景与测试分类矩阵

Cal.com 构建了覆盖时区数学边界、外部第三方日历集成与端到端预约的测试全矩阵：

| 测试层级 | 被测模块与用例范围 | 技术选型与执行环境 | 隔离机制与策略 |
| :--- | :--- | :--- | :--- |
| **Availability Math Unit** | 时区交集计算、缓冲时间（Buffer Time）叠加、槽位切分（Slot Splitting） | Vitest | 纯数学与时间戳纯函数计算，零外部依赖 |
| **DST Boundary Tests** | 夏令时切换凌晨（如 2:00 -> 3:00 跃迁）前后的可用槽位与预约生成 | Vitest + Luxon / Day.js | 固定时间桩与特定时区虚拟环境 |
| **Prisma Booking Integration** | 数据库并发锁定、防超卖（Double Booking）、悲观锁/排他约束 | Prisma + Test PostgreSQL | 独立测试数据库与事务自动重置 |
| **Calendar Sync Mock Tests** | Google Calendar API 响应超时、Token 过期刷新、同步冲突解决 | MSW (Mock Service Worker) | 拦截所有外部 OAuth 与日历 API 请求 |
| **Webhook Delivery Tests** | 预约成功事件派发、重试退避机制（Backoff）、签名验签 | Vitest + Supertest | 本地虚拟 HTTP 接收端验证签名 |
| **Booking Funnel E2E** | 真实浏览器中完成选择日期 -> 选择时间 -> 填写表单 -> 收到日历卡片 | Playwright | 自动化清除测试用户日历日程 |

---

## 3. 全模块自动化用例设计与实现详解

### 3.1 模块一：跨时区与多日程可用空闲时段（Availability Slots）交集计算测试

#### 3.1.1 业务功能与测试目标
发起人时区为 `America/New_York`，受访者时区为 `Asia/Shanghai`。发起人设置工作时间为工作日 09:00 - 17:00，且中间有 12:00 - 13:00 的日程占用。计算引擎必须输出：
1. 自动转换为 UTC 时间进行区间并集/交集计算。
2. 转换为受访者上海时区后，准确呈现对应的北京时间可选时段。
3. 扣除 15 分钟前置与后置缓冲时间（Buffer Time）。

#### 3.1.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';
import { getAvailableSlots } from '@calcom/core/getAvailableSlots';
import { Dayjs } from 'dayjs';

describe('Availability Engine Timezone Intersection Suite', () => {
  it('correctly intersects host availability and excludes busy slots across timezones', async () => {
    // 1. 发起人日程配置：纽约时区，工作时间 09:00 - 17:00
    const hostSchedule = {
      timeZone: 'America/New_York',
      workingHours: [
        { days: [1, 2, 3, 4, 5], startTime: 9 * 60, endTime: 17 * 60 } // 分钟表示
      ],
      busySlots: [
        // 纽约时间 13:00 - 14:00 有现存会议占用
        {
          start: '2026-10-15T13:00:00-04:00',
          end: '2026-10-15T14:00:00-04:00'
        }
      ]
    };

    // 2. 访问者时区：亚洲/上海
    const inviteeTimeZone = 'Asia/Shanghai';
    const dateQuery = '2026-10-15'; // 纽约对应日

    // 3. 计算 30 分钟长度的可用预约槽位
    const slots = await getAvailableSlots({
      schedule: hostSchedule,
      eventLength: 30,
      date: dateQuery,
      targetTimeZone: inviteeTimeZone
    });

    // 4. 断言返回的槽位中，绝对不包含对应纽约 13:00-14:00 (即北京时间次日凌晨 01:00-02:00) 的时间段
    const busySlotBeijingIso = '2026-10-16T01:00:00+08:00';
    const hasBusySlot = slots.some(slot => slot.time === busySlotBeijingIso);
    expect(hasBusySlot).toBe(false);

    // 验证正常工作时间对应槽位存在且数量符合预期
    expect(slots.length).toBeGreaterThan(10);
  });
});
```

---

### 3.2 模块二：夏令时（Daylight Saving Time）跳跃边界与容错测试

#### 3.2.1 业务功能与测试目标
在夏令时开始的当天（如欧美时间凌晨 02:00 直接跳到 03:00），系统不能生成不存在的 `02:30` 预约槽位；在冬令时结束当天（02:00 重复一次），不能生成重复冲突槽位。

#### 3.2.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';
import { calculateDaylightSlots } from '@calcom/core/timezones';

describe('Daylight Saving Time Jump Suite', () => {
  it('omits non-existent local hour during Spring Forward transition', () => {
    // 模拟 2026 年美国夏令时跳跃日（春季跳跃）
    const dstSunday = '2026-03-08';
    const timeZone = 'America/New_York';

    const generatedSlots = calculateDaylightSlots({
      date: dstSunday,
      timeZone,
      slotIntervalMinutes: 30
    });

    // 02:00 - 02:59 在当地物理上被跳过，绝对不可预约
    const hasInvalidHour = generatedSlots.some(slot => {
      const localHour = new Date(slot).toLocaleTimeString('en-US', { timeZone, hour: '2-digit', hour12: false });
      return localHour === '02';
    });

    expect(hasInvalidHour).toBe(false);
  });
});
```

---

### 3.3 模块三：并发高负载预约防超卖（Double Booking Prevention）数据库事务测试

#### 3.3.1 业务功能与测试目标
两个用户在同一秒对同一个空闲时段点击“确认预约”。数据库必须通过排他事务锁或唯一约束，保证只有一人成功创建 Booking，另一人必须收到 `409 Conflict` 友好提示，坚决杜绝超卖。

#### 3.3.2 测试用例实现
```typescript
import { describe, it, expect } from 'vitest';
import { prisma } from '@calcom/prisma';
import { createBooking } from '@calcom/features/bookings/lib/createBooking';

describe('Concurrency Double Booking Prevention Suite', () => {
  it('prevents double booking for identical slot via database transaction lock', async () => {
    const bookingSlot = {
      userId: 1,
      startTime: new Date('2026-11-01T10:00:00Z'),
      endTime: new Date('2026-11-01T10:30:00Z'),
      title: 'Product Consultation',
    };

    // 模拟两个用户完全并行的发起预约
    const results = await Promise.allSettled([
      createBooking({ ...bookingSlot, attendeeEmail: 'user_a@example.com' }),
      createBooking({ ...bookingSlot, attendeeEmail: 'user_b@example.com' }),
    ]);

    // 1. 验证结果分布：一个成功，一个被拒绝
    const fulfilled = results.filter(r => r.status === 'fulfilled');
    const rejected = results.filter(r => r.status === 'rejected');

    expect(fulfilled.length).toBe(1);
    expect(rejected.length).toBe(1);

    // 2. 验证数据库中物理记录严格仅有 1 条
    const dbCount = await prisma.booking.count({
      where: {
        userId: 1,
        startTime: bookingSlot.startTime,
      },
    });
    expect(dbCount).toBe(1);
  });
});
```

---

### 3.4 模块四：Playwright 预约漏斗端到端全链路（E2E）测试

#### 3.4.1 业务功能与测试目标
测试最终消费者从受邀请页面完成预约的全流程：
1. 访问用户公共预约链接 `http://localhost:3000/john/30min`。
2. 日历中点击可选日期，右侧抽屉滑动展示可选时间。
3. 选择 `10:00 AM`，输入姓名邮箱与备注，点击确认。
4. 页面成功跳转到 `Booking Confirmed` 页面并输出 `.ics` 下载卡片。

#### 3.4.2 测试用例实现
```typescript
import { test, expect } from '@playwright/test';

test.describe('Cal.com Booking Funnel E2E', () => {
  test('Complete public appointment booking flow from calendar to confirmation', async ({ page }) => {
    await page.goto('http://localhost:3000/pro-demo/30min');

    // 1. 等待日历组件挂载并点击下一个可用日期
    const availableDate = page.locator('[data-testid="day"][data-disabled="false"]').first();
    await expect(availableDate).toBeVisible({ timeout: 10000 });
    await availableDate.click();

    // 2. 选择具体的时钟槽位
    const timeSlot = page.locator('[data-testid="time"]').first();
    await expect(timeSlot).toBeVisible();
    const selectedTimeText = await timeSlot.innerText();
    await timeSlot.click();

    // 3. 填写预约人表单
    await page.fill('input[name="name"]', 'Alice Smith');
    await page.fill('input[name="email"]', 'alice@company.com');
    await page.fill('textarea[name="notes"]', 'Discuss partnership details');

    // 4. 点击 Confirm 提交
    await page.click('button[type="submit"]');

    // 5. 验证跳转至预约成功页面
    await expect(page.locator('[data-testid="success-page"]')).toBeVisible({ timeout: 15000 });
    await expect(page.locator('body')).toContainText('This meeting is scheduled');
    await expect(page.locator('body')).toContainText('Alice Smith');
  });
});
```

---

## 4. 测试工程化与基础设施

1. **时区测试全局基准与环境隔离**：
   * 在 Vitest 配置文件中，针对特定测试文件使用 `process.env.TZ = 'UTC'` 锁定基准，并在测试用例内部使用 `Intl.DateTimeFormat` 显式注入目标时区，避免受测试主机系统时区影响。
2. **MSW (Mock Service Worker) 模拟第三方日历 API**：
   * 将 Google Calendar、Outlook Graph API 的 OAuth 刷新与日程创建全部用 MSW 在网络层劫持，无需配置真实 Google 开发者账号即可进行完整故障注入（如测试 401 Token 失效自动刷新）。

---

## 5. 对同类复杂系统的参考价值

* **时间维度系统的数学求交与极端边界**：所有涉及排班、抢票、会议预订的系统，最大的漏洞往往出在夏令时切换与时区换算。在单元测试中必须建立针对关键时间节点的边界测试集（如 2 月 29 日闰年、12 月 31 日跨年、夏令时跳跃），才能保障系统全年无差错运行。
* **高并发核心资源的事务互斥**：对于不能超卖的核心资源，依赖应用层校验是不可靠的，必须在测试用例中引入真实的 `Promise.all` 并发压力测试，直接验证数据库层面的排他锁与唯一约束。
