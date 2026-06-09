# Glance 日历视图与节假日合成——代码深度分析

## 一、项目现状总览

> **关键结论：当前代码库中并不存在节假日（holiday）、iCal/ICS 日历事件订阅、以及多事件合并（merge/combine/overlap）的实现。** 日历组件目前只渲染一个纯日期网格，不含任何事件标记或节假日高亮。

如果你关注的是"日历视图与节假日合成的边界"，那么这个功能在当前版本中是**尚未实现的**。本文将系统梳理已有的代码结构，阐明日期生成、周起始、跨月溢出（spillover）、时区处理等基础逻辑，并指出未来接入节假日/事件时的自然扩展点。

涉及的核心文件：

| 文件 | 作用 |
|------|------|
| `internal/glance/widget-calendar.go` | 新版日历后端（Go），仅做配置解析与模板渲染 |
| `internal/glance/widget-old-calendar.go` | 旧版日历后端（Go），服务端生成日期数组 |
| `internal/glance/static/js/calendar.js` | **新版日历全部逻辑所在**：日期生成、月切换、动画 |
| `internal/glance/templates/calendar.html` | 新版日历模板 |
| `internal/glance/templates/old-calendar.html` | 旧版日历模板 |
| `internal/glance/widget-clock.go` | 时钟组件后端，包含时区校验逻辑 |
| `internal/glance/static/js/page.js` | 时钟前端，包含 `timeInZone()` 时区转换函数 |

---

## 二、两个日历版本的架构差异

### 2.1 新版日历（`type: calendar`）

架构上是**"后端给配置，前端算一切"**：

1. Go 后端只解析 `first-day-of-week`，把它转成整数 `FirstDay`（Sunday=0 … Saturday=6），写入 HTML data 属性。见 `internal/glance/widget-calendar.go` 第 21-41 行。
2. 模板只输出一个空的 `<div class="calendar" data-first-day-of-week="..."></div>`，不含任何日期。见 `internal/glance/templates/calendar.html`。
3. 页面加载时，前端 JS 通过 `setupCalendars()` 动态导入 `calendar.js`，**在浏览器里完全计算并渲染所有日期**。见 `internal/glance/static/js/page.js` 第 634-639 行。

优点：可交互（上/下月切换、回退到当月按钮、翻页动画）、不消耗服务端算力。
缺点：所有逻辑依赖浏览器 `Date` 对象，天然受客户端本地时区影响。

### 2.2 旧版日历（`type: calendar-legacy`，已废弃）

架构上是**"后端算好日期数组，模板直接渲染"**：

1. Go 后端在 `update()` 里调用 `newCalendar(time.Now(), widget.StartSunday)` 生成一个 21 天的日期数组（上一周 + 当前周 + 下一周）。见 `internal/glance/widget-old-calendar.go` 第 23-26 行。
2. 模板直接 `range .Calendar.Days` 输出。见 `internal/glance/templates/old-calendar.html` 第 28-32 行。

优点：确定性高，服务端控制时区。
缺点：无交互，只能看三周视图，代码已标注 TODO 要重构。

---

## 三、日期生成逻辑详解

### 3.1 新版日历（前端 JS）——完整月视图 6×7 = 42 格

核心函数：`Dates(firstDay)` 组件内部的 `updateFullMonth(now, newDate)`。
见 `internal/glance/static/js/calendar.js` 第 130-193 行。

关键常量：
```js
const FULL_MONTH_SLOTS = 7 * 6;  // 42 格，固定 6 行 × 7 列
const WEEKDAY_ABBRS = ["Su", "Mo", "Tu", "We", "Th", "Fr", "Sa"];
```

算法分四步：

**第 1 步：求"月首日星期几"**

```js
const firstWeekday = new Date(newDate.getFullYear(), newDate.getMonth(), 1).getDay();
```
`getDay()` 返回 JS 原生值：0=Sunday … 6=Saturday。

**第 2 步：计算上一个月需要"溢出"显示几天**

```js
const previousMonthSpilloverDays = (firstWeekday - firstDay + 7) % 7 || 7;
```

这是最容易出错的一行，展开讲：

- `firstDay` 是用户配置的周起始日（0=Sun … 6=Sat）。默认 `monday = 1`。
- `(firstWeekday - firstDay + 7) % 7` 解决了负数取模问题。
- `|| 7` 是特殊处理：**如果月首日恰好是周起始日，仍然要显示上一个月的整整一周（7 天）**，而不是 0 天。这保证了无论月首日落在哪，第一行都不会出现"全是当月"的情况，视觉上更统一，也与很多日历产品的行为一致。

> **边界判断关键点**：这里就是"跨月日期与当月日期的边界"。上一个月的日期会被 CSS 类 `calendar-spillover-date` 标记为暗色。当未来接入事件时，必须正确判断一个 `spillover` 格子到底属于哪个月，否则会把事件标错到相邻月。

**第 3 步：计算总天数并填充**

```js
const currentMonthDays = daysInMonth(newDate.getFullYear(), newDate.getMonth());
const nextMonthSpilloverDays = FULL_MONTH_SLOTS - (previousMonthSpilloverDays + currentMonthDays);
const previousMonthDays = daysInMonth(newDate.getFullYear(), newDate.getMonth() - 1);
```

- `daysInMonth(year, month)` 的实现很巧妙：`new Date(year, month + 1, 0).getDate()`，即下个月的第 0 天 = 当月最后一天。见 `internal/glance/static/js/calendar.js` 第 199-201 行。
- `new Date(year, month - 1, ...)` 即使 month 是 0（January）也没问题，JS 会自动回退到上一年 12 月。

然后按顺序填充 42 个格子：

```
[ 上一月的 spillover 天 ] → [ 当月 1..N 天 ] → [ 下一月的 spillover 天 ]
```

**第 4 步：标记"今天"**

```js
const isCurrentMonth = datesWithinSameMonth(now, newDate);
const currentDate = now.getDate();
```
仅当正在查看的月份就是当前系统月份时，才给当天加 `calendar-current-date` 高亮样式。

### 3.2 旧版日历（后端 Go）——三周视图 21 格

核心函数：`newCalendar(now time.Time, startSunday bool) *calendar`。
见 `internal/glance/widget-old-calendar.go` 第 42-82 行。

```go
weekday := now.Weekday()
if !startSunday {
    weekday = (weekday + 6) % 7   // Monday-first 时，把 Monday 从 1 映射为 0
}
startDaysFrom := now.Day() - int(weekday) - 7   // 再往前多退 7 天，即总共显示 3 周
```

然后 `for i := 0; i < 21; i++`，对 `day` 做越界处理：
- `< 1` 时回退到上一个月：`previousMonthDays + day`
- `> currentMonthDays` 时前进到下一个月：`day - currentMonthDays`

#### 重要修正：`daysInMonth` 使用 `time.UTC` **不构成时区问题**

`daysInMonth` 定义如下（`internal/glance/widget-old-calendar.go` 第 84-86 行）：
```go
func daysInMonth(m time.Month, year int) int {
    return time.Date(year, m+1, 0, 0, 0, 0, 0, time.UTC).Day()
}
```

之前的分析误认为这里用 `time.UTC` 会引入时区偏差——这是错误的。`daysInMonth` 的职责只是回答"某年某月有多少天"（28/29/30/31），这是一个**纯日历属性**，与时区完全无关。无论你在 UTC 还是 Asia/Shanghai，2024 年 2 月都是 29 天。用 `time.UTC` 只是为了创建一个临时 `time.Time` 对象来取 `Day()`，不会产生任何时区相关的偏差。

#### 真正的 Bug：ISOWeek year 与日历年混用

旧版日历的真实问题不在于时区，而在于年的来源：

```go
year, week := now.ISOWeek()   // ← ISO 周年，不一定等于日历年！
...
currentMonthDays := daysInMonth(now.Month(), year)   // ← 用 ISO 周年 + 日历月
...
if previousMonthNumber := now.Month() - 1; previousMonthNumber < 1 {
    previousMonthDays = daysInMonth(12, year-1)      // ← 跨年时 year-1 也是错的
}
...
CurrentYear: year,   // ← 界面上显示的年份也是 ISO 周年
```

`time.Time.ISOWeek()` 返回的是 ISO 8601 周编号体系下的"周年"和"周号"，规则是：
- 每周从周一开始
- 一年的第 1 周是包含该年第一个周四的那一周

这意味着**1 月初的几天可能属于上一年的 ISO 周，12 月底的几天可能属于下一年的 ISO 周**。例如：
- 2023-01-01（周日）：`ISOWeek()` 返回 `(2022, 52)`，但 `now.Month()` = January，`now.Year()` = 2023
- 2024-12-30（周一）：`ISOWeek()` 返回 `(2025, 1)`，但 `now.Month()` = December，`now.Year()` = 2024

当前代码把 ISO 周年 `year` 和日历月 `now.Month()` 直接混用，理论上会算错天数。不过由于：
- 只有 1 月和 12 月才会出现 ISO 周年 ≠ 日历年
- 1 月、12 月的天数固定为 31 天，与年份无关
- 2 月的闰年差异不可能出现在 ISO 周年错配的场景里

所以**这个 bug 在实际运行中几乎不会产生可观察的错误**，但从代码正确性角度，`year` 应该取 `now.Year()` 而不是 `now.ISOWeek()` 的返回值。界面上显示的 `CurrentYear` 同样应该用日历年。

### 3.3 旧版日历：`start-sunday`、ISOWeek 与模板显示的一致性问题

这是旧版日历中最复杂、最容易被忽略的一组边界问题。核心矛盾是：**`ISOWeek()` 对"一周"的定义（周一起始）与 `start-sunday: true` 时网格对"一周"的定义（周日起始）不一致，并且代码没有做任何适配。**

#### 3.3.1 三者的数据流与职责

先看 `newCalendar()` 和模板中三个关键变量的来源：

| 变量 | 来源 | 定义 | 受 `startSunday` 影响？ |
|------|------|------|------------------------|
| `CurrentWeekNumber` | `now.ISOWeek()` 返回值的第二个 | **ISO 8601 周号**：每周从周一开始，第 1 周是包含当年第一个周四的那一周 | ❌ 完全不受影响 |
| `weekday`（用于计算 `startDaysFrom`） | `now.Weekday()`，根据 `startSunday` 做偏移 | `startSunday=true`：Sunday=0…Saturday=6；`startSunday=false`：Monday=0…Sunday=6 | ✅ 直接影响 |
| 星期表头顺序 | 模板中 `{{ if .StartSunday }}` 分支 | 决定 Su 放在最前还是最后 | ✅ 直接影响 |

注意一个关键事实：`year, week := now.ISOWeek()` 这一行写在 `startSunday` 条件判断的**最前面**，完全不考虑用户选择了哪种周起始日。

#### 3.3.2 周一起始模式（`start-sunday: false`，默认）——基本一致

在默认模式下，ISO 周的定义与网格的定义是对齐的：两者都把周一视为一周的第一天。

举个具体例子，假设 `now = 2024-06-09`（周日）：
- `now.ISOWeek()` = `(2024, 23)`，即 ISO 第 23 周（2024-06-03 周一 ~ 2024-06-09 周日）
- `startSunday = false`，所以 `weekday = (0 + 6) % 7 = 6`（周日在周一起始体系下排第 6 位）
- `startDaysFrom = 9 - 6 - 7 = -4`，从 5 月 28 日开始取 21 天

生成的 3 行 × 7 列网格（表头：Mo Tu We Th Fr Sa Su）：

| 行 | 日期范围 | 对应 ISO 周 |
|----|----------|-------------|
| 第 1 行（上一周） | 5/28 ~ 6/2 | ISO W22 |
| 第 2 行（本周） | 6/3 ~ 6/9 | ISO W23 ✅ 与 `CurrentWeekNumber=23` 完全一致 |
| 第 3 行（下一周） | 6/10 ~ 6/16 | ISO W24 |

此时用户看到 "Week 23" 和中间一行 6/3~6/9，两者完全吻合，没有任何问题。

#### 3.3.3 周日起始模式（`start-sunday: true`）——周号与网格中间行错位

当用户设置 `start-sunday: true` 时，问题开始显现。仍然用 `now = 2024-06-09`（周日）：
- `now.ISOWeek()` = `(2024, 23)`，ISO W23 仍然是 6/3（周一）~ 6/9（周日）——这个定义不变
- `startSunday = true`，所以 `weekday` 保持 Go 原生值 `0`（Sunday=0）
- `startDaysFrom = 9 - 0 - 7 = 2`，从 6 月 2 日开始取 21 天

生成的 3 行 × 7 列网格（表头：Su Mo Tu We Th Fr Sa）：

| 行 | 日期范围 | 对应 ISO 周 |
|----|----------|-------------|
| 第 1 行（上一周） | 6/2（Su）~ 6/8（Sa） | 6/2 属于 ISO W22，6/3~6/8 属于 ISO W23 |
| 第 2 行（本周） | 6/9（Su）~ 6/15（Sa） | **6/9 属于 ISO W23，6/10~6/15 属于 ISO W24** |
| 第 3 行（下一周） | 6/16（Su）~ 6/22（Sa） | ISO W25 |

**问题所在**：界面右上角显示 `Week 23`，暗示中间这一整行都是第 23 周。但实际上中间行只有第 1 天（6 月 9 日，周日）属于 ISO W23，其余 6 天全部属于 ISO W24。用户会被误导。

#### 3.3.4 更极端的边界：ISO 周跨年 + `start-sunday: true`

用 `now = 2023-01-01`（周日）验证：
- `now.ISOWeek()` = `(2022, 52)`，ISO W52（2022-12-26 周一 ~ 2023-01-01 周日）
- `startSunday = true`，`weekday = 0`
- `startDaysFrom = 1 - 0 - 7 = -6`，从 2022 年 12 月 26 日开始（12 月有 31 天，`-6 + 31 = 25`？不对，看代码：`day = previousMonthDays + day = 31 + (-6) = 25`，即从 12 月 25 日开始）

网格（表头：Su Mo Tu We Th Fr Sa）：

| 行 | 日期范围 | 对应 ISO 周 |
|----|----------|-------------|
| 第 1 行 | 12/25（Su）~ 12/31（Sa） | 12/25~12/25=ISO W51，12/26~12/31=ISO W52 |
| 第 2 行 | 1/1（Su）~ 1/7（Sa） | **1/1 属于 ISO W52（2022），1/2~1/7 属于 ISO W1（2023）** |
| 第 3 行 | 1/8（Su）~ 1/14（Sa） | ISO W2（2023） |

同时界面上还会显示：
- 左侧：`January`（月份名，来自 `now.Month().String()`）
- 右上：`Week 52` + `2022`（ISO 周年与周号，来自 `now.ISOWeek()`）

于是出现三重错位：
1. 月份名是 `January`（日历年 2023 年 1 月），但年份显示 `2022`（ISO 周年）
2. `Week 52` 对应的 ISO W52 范围是 12/26~1/1，但中间行显示的是 1/1~1/7
3. 中间行本身跨越了两个 ISO 周（W52 和 W1），甚至跨越了两个 ISO 周年（2022 和 2023）

#### 3.3.5 根因总结

旧版日历的周号显示存在**四层定义不一致**：

1. **`ISOWeek()` 周号的定义**：周一起始，ISO 8601 标准
2. **`startSunday=true` 时网格"一周"的定义**：周日起始
3. **`CurrentYear` 的来源**：ISO 周年，可能与日历年不同
4. **`CurrentMonthName` 的来源**：日历月，来自 `now.Month()`

在 `start-sunday: false` 时，①和②恰好对齐，加上 1/12 月天数恒为 31 的巧合，使问题不可见；但在 `start-sunday: true` 时，这四层定义的冲突完全暴露。

#### 3.3.6 如果要修复

最小修复思路：
- `CurrentYear` 改为 `now.Year()`（日历年）
- `CurrentWeekNumber` 不再直接取 `ISOWeek()`，而是根据 `startSunday` 自行计算：
  - `startSunday=false`：可以继续用 `ISOWeek()`，因为定义一致
  - `startSunday=true`：需要实现一套"周日起始"的周编号算法（北美常用体系：第 1 周是包含 1 月 1 日的那一周），或直接不显示周号
- 或者更简单：当 `startSunday=true` 时，模板中隐藏周号显示，避免误导

---

## 四、周起始（First Day of Week）处理

两个版本实现方式不同：

### 4.1 新版日历

配置字段 `first-day-of-week`，可选值 `sunday` 到 `saturday`，默认 `monday`。
见 `internal/glance/widget-calendar.go` 第 11-37 行。

映射表：
```go
var calendarWeekdaysToInt = map[string]time.Weekday{
    "sunday":    time.Sunday,    // 0
    "monday":    time.Monday,    // 1
    ...
    "saturday":  time.Saturday,  // 6
}
```

前端拿到这个整数后有**两处**使用：

1. **星期表头顺序**：`WEEKDAY_ABBRS[(firstDay + i) % 7]`，见 `internal/glance/static/js/calendar.js` 第 184-186 行。
2. **上月 spillover 天数**：就是上面 3.1 节的 `previousMonthSpilloverDays` 公式。

### 4.2 旧版日历

配置字段 `start-sunday`（布尔），默认 `false`（即周一起始）。
见 `internal/glance/widget-old-calendar.go` 第 14 行。

模板里用 `{{ if .StartSunday }}` 决定是否把 Su 放在最前面。见 `internal/glance/templates/old-calendar.html` 第 13-26 行。

---

## 五、时区处理

### 5.1 日历本身**没有显式时区处理**

这是一个非常重要的观察：

- **新版日历**完全依赖浏览器 `new Date()`，即客户端本地时区。对于跨时区用户（比如服务器在 UTC，人在 Asia/Shanghai），"今天"的判定是以用户电脑为准——这通常是正确的行为，但也意味着无法通过服务端配置强制统一时区。
- **旧版日历**使用 Go 服务端的 `time.Now()`，即**服务端进程所在的本地时区**（取决于操作系统的 `TZ` 环境变量或系统设置）。如果用户和服务器不在同一时区，看到的"今天"、当前周号、月份名称可能与用户本地时间不一致。注意：`daysInMonth` 里的 `time.UTC` 与时区问题无关（见 3.2 节修正）。

### 5.2 项目中其他地方的时区实现可供参考

日历本身没做，但 **clock 时钟组件**有完整的时区处理，未来日历如果要加时区可直接复用思路。

**后端校验**（Go）：
```go
if _, err := time.LoadLocation(widget.Timezones[t].Timezone); err != nil {
    return fmt.Errorf("invalid timezone '%s': %v", ...)
}
```
见 `internal/glance/widget-clock.go` 第 31-38 行。

**前端转换**（JS）：
```js
function timeInZone(now, zone) {
    timeInZone = new Date(now.toLocaleString('en-US', { timeZone: zone }));
    const diffInMinutes = Math.round((timeInZone.getTime() - now.getTime()) / 1000 / 60);
    return { time: timeInZone, diffInMinutes: diffInMinutes };
}
```
见 `internal/glance/static/js/page.js` 第 532-546 行。

这个函数的技巧是利用 `toLocaleString` 的 `{ timeZone }` 选项把一个 `Date` 对象"看起来变成"目标时区的本地时间，再重新解析成 Date。虽然不够优雅，但在浏览器原生 API 里是最通用的做法。

---

## 六、事件合并与节假日——现状与扩展点

### 6.1 现状：完全没有实现

全局搜索 `holiday`、`ical`、`ics`、`event`、`merge`、`overlap`、`conflict` 等关键词，在日历相关代码中**零命中**。`go.mod` 里也没有引入任何 iCal 解析库。

### 6.2 如果要接入，自然扩展点在哪

基于现有代码结构，建议的接入路径：

**后端（Go）侧**：在 `internal/glance/widget-calendar.go` 中扩展：
- 新增配置字段，如 `holiday-providers`、`ical-urls`、`timezone`
- 引入 iCal 解析库（如 `github.com/arran4/golang-ical`）拉取并解析 ICS
- 把事件按日期（YYYY-MM-DD）聚合成一个 `map[string][]CalendarEvent`，JSON 序列化后通过模板 data 属性下发给前端（类似 `data-first-day-of-week` 的做法）

**前端（JS）侧**：在 `internal/glance/static/js/calendar.js` 的 `updateFullMonth()` 中扩展：
- 目前每个格子只 `.text(i)` 写日期数字；可以在这一步查找该日期对应的事件数组
- **"事件合并"的边界判断**：一个日期格子可能同时有 `spillover`（属于相邻月但显示在本月视图里）、`current-date`（今天）、以及多个事件。优先级建议：
  1. 先确定该格子的**归属日期**（考虑 spillover 和时区，是上月/当月/下月的哪一天）
  2. 再按归属日期去事件表里查
  3. 同一天多个事件的合并策略：按开始时间排序，超过 N 个时折叠显示 "+N more"

**CSS 侧**：`internal/glance/static/css/widget-calendar.css` 中 `.calendar-date` 已经是 `position: relative`，可以直接在里面 `::after` 画事件小圆点，或追加事件徽标元素。

---

## 七、自动更新（跨零点滚动）

新版日历有一个 `autoAdvanceNow()` 定时器：
```js
const autoAdvanceNow = () => {
    advanceTimeTicker = setTimeout(() => {
        update(now = new Date());
        autoAdvanceNow();
    }, msTillNextDay());
};
```
见 `internal/glance/static/js/calendar.js` 第 51-57 行。

`msTillNextDay()` 计算距离下一个本地零点还有多少毫秒，确保"今天"的高亮会在零点自动切换。但代码里有个 TODO：`// TODO: don't auto advance if looking at a different month`——目前即便你在看上/下个月，零点时也会强行跳回当月。

---

## 八、总结

| 能力 | 新版日历（前端） | 旧版日历（后端） | 备注 |
|------|------------------|------------------|------|
| 日期生成 | JS `Date`，42 格月视图 | Go `time.Time`，21 格三周视图 | |
| 周起始配置 | 7 种可选，默认周一 | 二选一（周日/周一），默认周一 | |
| 跨月 spillover 边界 | `(firstWeekday - firstDay + 7) % 7 \|\| 7` | `startDaysFrom - weekday - 7` | 新版的 `\|\| 7` 是关键边界 |
| 时区处理 | 隐式使用浏览器本地时区 | 隐式使用服务端本地时区 | **均无显式配置**。旧版 `daysInMonth` 中的 `time.UTC` 不构成时区问题 |
| 周号与网格一致性 | 不显示周号 | `start-sunday: true` 时周号与中间行错位 | 详见 3.3 节：ISOWeek 定义（周一起始）与网格（周日起始）不一致，跨年时还伴随 ISO 周年、日历年、日历月三重错位 |
| 其他已知 Bug | 无 | ISOWeek year 与日历年混用 | 实际影响极小，但代码不正确（见 3.2 节） |
| 事件/节假日 | ❌ 未实现 | ❌ 未实现 | 需按 6.2 节扩展 |
| 事件合并 | ❌ 未实现 | ❌ 未实现 | 关键边界：先确定格子归属日期，再查事件 |
| 自动跨天 | ✅ 零点定时器 | ❌ 依赖整点缓存刷新 | |
