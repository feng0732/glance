# Glance 日历视图与节假日合成——代码深度分析

## 一、项目现状总览

> **关键结论：当前代码库中并不存在节假日（holiday）、iCal/ICS 日历事件订阅、以及多事件合并（merge/combine/overlap）的实现。** 日历组件目前只渲染一个纯日期网格，不含任何事件标记或节假日高亮。

如果你关注的是"日历视图与节假日合成的边界"，那么这个功能在当前版本中是**尚未实现的**。本文将系统梳理已有的代码结构，阐明日期生成、周起始、跨月溢出（spillover）、时区处理等基础逻辑，并指出未来接入节假日/事件时的自然扩展点。

涉及的核心文件：

| 文件 | 作用 |
|------|------|
| [widget-calendar.go](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-calendar.go) | 新版日历后端（Go），仅做配置解析与模板渲染 |
| [widget-old-calendar.go](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-old-calendar.go) | 旧版日历后端（Go），服务端生成日期数组 |
| [calendar.js](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/calendar.js) | **新版日历全部逻辑所在**：日期生成、月切换、动画 |
| [calendar.html](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/templates/calendar.html) | 新版日历模板 |
| [old-calendar.html](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/templates/old-calendar.html) | 旧版日历模板 |
| [widget-clock.go](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-clock.go) | 时钟组件后端，包含时区校验逻辑 |
| [page.js](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/page.js) | 时钟前端，包含 `timeInZone()` 时区转换函数 |

---

## 二、两个日历版本的架构差异

### 2.1 新版日历（`type: calendar`）

架构上是**"后端给配置，前端算一切"**：

1. Go 后端只解析 `first-day-of-week`，把它转成整数 `FirstDay`（Sunday=0 … Saturday=6），写入 HTML data 属性。见 [widget-calendar.go#L21-L41](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-calendar.go#L21-L41)。
2. 模板只输出一个空的 `<div class="calendar" data-first-day-of-week="..."></div>`，不含任何日期。见 [calendar.html#L1-L7](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/templates/calendar.html#L1-L7)。
3. 页面加载时，前端 JS 通过 `setupCalendars()` 动态导入 `calendar.js`，**在浏览器里完全计算并渲染所有日期**。见 [page.js#L634-L639](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/page.js#L634-L639)。

优点：可交互（上/下月切换、回退到当月按钮、翻页动画）、不消耗服务端算力。
缺点：所有逻辑依赖浏览器 `Date` 对象，天然受客户端本地时区影响。

### 2.2 旧版日历（`type: calendar-legacy`，已废弃）

架构上是**"后端算好日期数组，模板直接渲染"**：

1. Go 后端在 `update()` 里调用 `newCalendar(time.Now(), widget.StartSunday)` 生成一个 21 天的日期数组（上一周 + 当前周 + 下一周）。见 [widget-old-calendar.go#L23-L26](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-old-calendar.go#L23-L26)。
2. 模板直接 `range .Calendar.Days` 输出。见 [old-calendar.html#L28-L32](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/templates/old-calendar.html#L28-L32)。

优点：确定性高，服务端控制时区。
缺点：无交互，只能看三周视图，代码已标注 TODO 要重构。

---

## 三、日期生成逻辑详解

### 3.1 新版日历（前端 JS）——完整月视图 6×7 = 42 格

核心函数：`Dates(firstDay)` 组件内部的 `updateFullMonth(now, newDate)`。
见 [calendar.js#L130-L193](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/calendar.js#L130-L193)。

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

- `daysInMonth(year, month)` 的实现很巧妙：`new Date(year, month + 1, 0).getDate()`，即下个月的第 0 天 = 当月最后一天。见 [calendar.js#L199-L201](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/calendar.js#L199-L201)。
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
见 [widget-old-calendar.go#L42-L82](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-old-calendar.go#L42-L82)。

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

注意旧版的 `daysInMonth()` 也是同样的"下个月第 0 天"技巧，但用的是 `time.UTC` 固定时区。见 [widget-old-calendar.go#L84-L86](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-old-calendar.go#L84-L86)。

---

## 四、周起始（First Day of Week）处理

两个版本实现方式不同：

### 4.1 新版日历

配置字段 `first-day-of-week`，可选值 `sunday` 到 `saturday`，默认 `monday`。
见 [widget-calendar.go#L11-L37](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-calendar.go#L11-L37)。

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

1. **星期表头顺序**：`WEEKDAY_ABBRS[(firstDay + i) % 7]`，见 [calendar.js#L184-L186](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/calendar.js#L184-L186)。
2. **上月 spillover 天数**：就是上面 3.1 节的 `previousMonthSpilloverDays` 公式。

### 4.2 旧版日历

配置字段 `start-sunday`（布尔），默认 `false`（即周一起始）。
见 [widget-old-calendar.go#L14](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-old-calendar.go#L14)。

模板里用 `{{ if .StartSunday }}` 决定是否把 Su 放在最前面。见 [old-calendar.html#L13-L26](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/templates/old-calendar.html#L13-L26)。

---

## 五、时区处理

### 5.1 日历本身**没有显式时区处理**

这是一个非常重要的观察：

- **新版日历**完全依赖浏览器 `new Date()`，即客户端本地时区。对于跨时区用户（比如服务器在 UTC，人在 Asia/Shanghai），"今天"的判定是以用户电脑为准。
- **旧版日历**使用 Go 服务端的 `time.Now()`，然后 `daysInMonth()` 里用 `time.UTC` 计算。这在极端的时区边界（比如服务器在 UTC+14、客户端在 UTC-12）会出现"服务端和客户端认为'今天'不是同一天"的问题。

### 5.2 项目中其他地方的时区实现可供参考

日历本身没做，但 **clock 时钟组件**有完整的时区处理，未来日历如果要加时区可直接复用思路。

**后端校验**（Go）：
```go
if _, err := time.LoadLocation(widget.Timezones[t].Timezone); err != nil {
    return fmt.Errorf("invalid timezone '%s': %v", ...)
}
```
见 [widget-clock.go#L31-L38](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-clock.go#L31-L38)。

**前端转换**（JS）：
```js
function timeInZone(now, zone) {
    timeInZone = new Date(now.toLocaleString('en-US', { timeZone: zone }));
    const diffInMinutes = Math.round((timeInZone.getTime() - now.getTime()) / 1000 / 60);
    return { time: timeInZone, diffInMinutes: diffInMinutes };
}
```
见 [page.js#L532-L546](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/page.js#L532-L546)。

这个函数的技巧是利用 `toLocaleString` 的 `{ timeZone }` 选项把一个 `Date` 对象"看起来变成"目标时区的本地时间，再重新解析成 Date。虽然不够优雅，但在浏览器原生 API 里是最通用的做法。

---

## 六、事件合并与节假日——现状与扩展点

### 6.1 现状：完全没有实现

全局搜索 `holiday`、`ical`、`ics`、`event`、`merge`、`overlap`、`conflict` 等关键词，在日历相关代码中**零命中**。`go.mod` 里也没有引入任何 iCal 解析库。

### 6.2 如果要接入，自然扩展点在哪

基于现有代码结构，建议的接入路径：

**后端（Go）侧**：在 [widget-calendar.go](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/widget-calendar.go) 中扩展：
- 新增配置字段，如 `holiday-providers`、`ical-urls`、`timezone`
- 引入 iCal 解析库（如 `github.com/arran4/golang-ical`）拉取并解析 ICS
- 把事件按日期（YYYY-MM-DD）聚合成一个 `map[string][]CalendarEvent`，JSON 序列化后通过模板 data 属性下发给前端（类似 `data-first-day-of-week` 的做法）

**前端（JS）侧**：在 [calendar.js](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/calendar.js) 的 `updateFullMonth()` 中扩展：
- 目前每个格子只 `.text(i)` 写日期数字；可以在这一步查找该日期对应的事件数组
- **"事件合并"的边界判断**：一个日期格子可能同时有 `spillover`（属于相邻月但显示在本月视图里）、`current-date`（今天）、以及多个事件。优先级建议：
  1. 先确定该格子的**归属日期**（考虑 spillover 和时区，是上月/当月/下月的哪一天）
  2. 再按归属日期去事件表里查
  3. 同一天多个事件的合并策略：按开始时间排序，超过 N 个时折叠显示 "+N more"

**CSS 侧**：[widget-calendar.css](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/css/widget-calendar.css) 中 `.calendar-date` 已经是 `position: relative`，可以直接在里面 `::after` 画事件小圆点，或追加事件徽标元素。

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
见 [calendar.js#L51-L57](file:///d:/fz/0601/solo-dogfeeding/code/139-glance/internal/glance/static/js/calendar.js#L51-L57)。

`msTillNextDay()` 计算距离下一个本地零点还有多少毫秒，确保"今天"的高亮会在零点自动切换。但代码里有个 TODO：`// TODO: don't auto advance if looking at a different month`——目前即便你在看上/下个月，零点时也会强行跳回当月。

---

## 八、总结

| 能力 | 新版日历（前端） | 旧版日历（后端） | 备注 |
|------|------------------|------------------|------|
| 日期生成 | JS `Date`，42 格月视图 | Go `time.Time`，21 格三周视图 | |
| 周起始配置 | 7 种可选，默认周一 | 二选一（周日/周一），默认周一 | |
| 跨月 spillover 边界 | `(firstWeekday - firstDay + 7) % 7 \|\| 7` | `startDaysFrom - weekday - 7` | 新版的 `\|\| 7` 是关键边界 |
| 时区处理 | 隐式使用浏览器本地时区 | `time.Now()` + `time.UTC` 混用 | **均无显式配置** |
| 事件/节假日 | ❌ 未实现 | ❌ 未实现 | 需按 6.2 节扩展 |
| 事件合并 | ❌ 未实现 | ❌ 未实现 | 关键边界：先确定格子归属日期，再查事件 |
| 自动跨天 | ✅ 零点定时器 | ❌ 依赖整点缓存刷新 | |
