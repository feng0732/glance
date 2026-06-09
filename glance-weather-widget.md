# Weather Widget 定位与预报聚合分析

本文档按代码执行顺序，解析天气 Widget 的四大核心流程：位置解析、单位转换、时区对齐、未来时段处理。

---

## 一、位置解析（Location Parsing）

### 1.1 用户输入格式

用户在 YAML 配置中通过 `location` 字段指定位置，支持三种格式：

| 格式          | 示例                | 含义                   |
|---------------|---------------------|------------------------|
| 单段          | `Beijing`           | 仅城市名               |
| 两段          | `London, UK`        | 城市 + 国家            |
| 三段          | `Brooklyn, NY, US`  | 城市 + 行政区 + 国家   |

### 1.2 `parsePlaceName` 函数

[widget-weather.go#L157-L169](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L157-L169)

该函数将用户输入拆分为 **查询关键词** 和 **行政区过滤词**，同时展开国家缩写：

```
输入拆分规则（按逗号分割）：
┌───────────────┬──────────────────────────┬─────────────────────┐
│ 分段数        │ 查询关键词 (location)    │ 过滤词 (area)       │
├───────────────┼──────────────────────────┼─────────────────────┤
│ 1 段          │ 原样返回                 │ ""（空）             │
│ 2 段          │ parts[0] + expand(parts[1]) │ ""（空）         │
│ 3 段          │ parts[0] + expand(parts[2]) │ parts[1]         │
└───────────────┴──────────────────────────┴─────────────────────┘
```

国家缩写展开表（`commonCountryAbbreviations`）：
- `US` / `USA` → `United States`
- `UK` → `United Kingdom`

**设计意图**：Open-Meteo 地理编码 API 不接受国家缩写，三段式时中间段（如 NY）为行政区，不作为查询词而是作为结果过滤条件。

### 1.3 `fetchOpenMeteoPlaceFromName` 函数

[widget-weather.go#L171-L211](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L171-L211)

调用 Open-Meteo Geocoding API：`https://geocoding-api.open-meteo.com/v1/search`

参数：`count=20&language=en&format=json`

结果匹配策略：
1. **无过滤词（area 为空）**：直接取返回结果第 1 条 `Results[0]`
2. **有过滤词（area 非空）**：遍历 Results，按不区分大小写匹配 `admin1` 字段（即 API 返回的 Area），找到第一个匹配项

匹配成功后，将 `place.Timezone` 字符串通过 `time.LoadLocation()` 加载为 `*time.Location`，存入 `place.location` 字段（未导出，仅供内部使用）。

---

## 二、单位转换（Unit Conversion）

### 2.1 单位初始化

[widget-weather.go#L50-L54](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L50-L54)

`initialize()` 中设置默认单位：
- 空值 → `"metric"`（摄氏度）
- 仅接受 `"metric"` 或 `"imperial"`，否则报错

### 2.2 API 请求单位映射

[widget-weather.go#L214-L231](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L214-L231)

在构造天气 API 请求时，将用户单位转换为 Open-Meteo 参数：

```
Widget 配置    →    Open-Meteo 参数
"metric"       →    temperature_unit=celsius
"imperial"     →    temperature_unit=fahrenheit
```

请求通过 `query.Add("temperature_unit", temperatureUnit)` 传递，API 直接返回对应单位的数值，Widget 无需二次换算。

### 2.3 模板渲染

[weather.html#L6](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/templates/weather.html#L6)

模板根据 Units 字段显示不同的度数符号：
- `metric` → 显示 `°C`
- `imperial` → 显示 `°F`

---

## 三、时区对齐（Timezone Alignment）

### 3.1 时区信息来源

时区字符串来自地理编码 API 返回的 `place.Timezone`，如 `"Asia/Shanghai"`、`"America/New_York"`。

在 `fetchOpenMeteoPlaceFromName` 末尾加载：
```go
loc, err := time.LoadLocation(place.Timezone)
place.location = loc
```

### 3.2 API 层面的时区对齐

[widget-weather.go#L225-L227](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L225-L227)

请求天气 API 时显式指定时区参数：
```
query.Add("timeformat", "unixtime")   // 时间戳格式
query.Add("timezone", place.Timezone) // 目标地点时区
query.Add("forecast_days", "1")       // 仅取未来1天
```

这确保 API 返回的 hourly 数据数组索引 0~23 恰好对应目标地点本地时间的 0 点 ~ 23 点。

### 3.3 代码层面的时区对齐

[widget-weather.go#L240-L244](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L240-L244)

所有时间计算均通过 `time.Now().In(place.location)` 转换到目标地点时区：

```go
now := time.Now().In(place.location)               // 当前时间（目标时区）
currentBar := now.Hour() / 2                        // 当前处于哪一列（0~11）

// 日出时间戳 → 目标时区时间 → 小时数 → 列索引
sunriseBar := (time.Unix(sunrise, 0).In(place.location).Hour()) / 2

// 日落时间戳 → 目标时区时间 → 小时数 -1 → 列索引（向前偏移1小时避免边界问题）
sunsetBar  := (time.Unix(sunset, 0).In(place.location).Hour() - 1) / 2
```

`sunsetBar` 做 `-1` 处理后若小于 0，则修正为 0。

---

## 四、未来时段处理（Forecast Aggregation）

### 4.1 数据模型

Widget 展示 **12 个双小时时段**（2-hour buckets），对应一天 24 小时。每个时段为 `weatherColumn`：

```go
type weatherColumn struct {
    Temperature      int       // 该时段温度（整数）
    Scale            float64   // 温度相对高度（0.0 ~ 1.0，用于柱状图渲染）
    HasPrecipitation bool      // 是否有降水概率
}
```

### 4.2 数据完整性检查与留空路径

[widget-weather.go#L240-L284](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L240-L284)

在聚合之前，代码先初始化了一个空切片和三个关键列索引：

```go
now := time.Now().In(place.location)
bars := make([]weatherColumn, 0, 24)        // 空切片，容量 24
currentBar := now.Hour() / 2
sunriseBar := (time.Unix(int64(responseJson.Daily.Sunrise[0]), 0).In(place.location).Hour()) / 2
sunsetBar  := (time.Unix(int64(responseJson.Daily.Sunset[0]),  0).In(place.location).Hour() - 1) / 2
```

真正的聚合逻辑被包裹在一层长度守卫中：

```go
if len(responseJson.Hourly.Temperature) == 24 {
    // ... 执行 24→12 聚合，向 bars 中 append 12 个 weatherColumn
}
```

**留空路径（静默降级）**：当 API 返回的 `Hourly.Temperature` 数组长度不等于 24 时（例如请求接近午夜、API 异常、时区边界问题导致只返回部分时段数据），整个聚合块被跳过，`bars` 切片保持为空（长度 0）。最终 `weather.Columns` 是空切片，模板中 `{{ range $i, $column := .Weather.Columns }}` 不会产生任何迭代输出，预报列区域完全空白，但天气状况文字和体感温度仍能正常显示。

该行为是一种**静默降级策略**：不报错、不填充占位数据，直接跳过渲染。

---

### 4.2.1 边界风险全景：数组长度不一致与越界访问

`fetchWeatherForOpenMeteoPlace` 函数中存在四处数组索引访问，但仅有一处做了长度校验，以下按执行顺序逐一分析：

#### 风险 ①：Daily.Sunrise / Daily.Sunset 空数组 panic

[widget-weather.go#L243-L244](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L243-L244)

```go
sunriseBar := (time.Unix(int64(responseJson.Daily.Sunrise[0]), 0).In(place.location).Hour()) / 2
sunsetBar  := (time.Unix(int64(responseJson.Daily.Sunset[0]),  0).In(place.location).Hour() - 1) / 2
```

- **触发条件**：API 返回了 `daily` 字段但 `sunrise` 或 `sunset` 数组为空（长度 0）
- **后果**：Go 运行时 `index out of range` panic，整个 Widget 更新 goroutine 崩溃，`widget.Weather` 保持 `nil`，下次 `update()` 重新尝试
- **现实概率**：极低。Open-Meteo 在 `forecast_days=1` 时通常返回恰好 1 条日出/日落数据，但理论上极区极昼/极夜、或 API 故障时可能返回空数组

#### 风险 ②：温度与降水概率长度不一致导致 panic（核心风险）

[widget-weather.go#L250-L265](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L250-L265)

```go
if len(responseJson.Hourly.Temperature) == 24 {    // 仅校验 Temperature
    t := responseJson.Hourly.Temperature
    p := responseJson.Hourly.PrecipitationProbability   // ← 未校验 p 的长度

    for i := 0; i < 24; i += 2 {
        // ...
        precipitations[i/2] = (p[i]+p[i+1])/2 > 75  // ← 直接访问 p[i] 和 p[i+1]
    }
}
```

- **触发条件**：`len(Temperature) == 24` 通过了守卫，但 `len(PrecipitationProbability) < 24`（例如 `nil`、长度 1、长度 23 等）
- **后果**：`p[i]` 或 `p[i+1]` 越界 panic，整个 goroutine 崩溃
- **现实场景**：
  - API 返回结构异常，`hourly` 对象存在但缺少 `precipitation_probability` 字段（此时 p 为 nil 切片，长度 0）
  - Open-Meteo 部分数据缺失，降水概率只返回了部分时段
  - 两个字段长度分别为 24 和 N（N < 24），代码无任何防御

#### 风险 ③：Temperature 长度校验不包含上界

[widget-weather.go#L250](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L250)

```go
if len(responseJson.Hourly.Temperature) == 24 {
    for i := 0; i < 24; i += 2 {
        temperatures[i/2] = int(math.Round((t[i] + t[i+1]) / 2))
```

如果 `len(Temperature) > 24`，守卫不成立，bars 留空（静默降级，安全）；如果恰好等于 24，循环中 `i+1` 最大为 23，不会越界。此处逻辑是安全的。

#### 风险全景汇总表

| # | 代码位置 | 访问对象 | 有无长度守卫 | 异常场景 | 后果 |
|---|---------|---------|------------|---------|------|
| ① | L243 | `Daily.Sunrise[0]` | 无 | sunrise 数组为空 | panic |
| ② | L244 | `Daily.Sunset[0]` | 无 | sunset 数组为空 | panic |
| ③ | L261 | `Temperature[i]` / `[i+1]` | 有（`==24`） | 长度 != 24 | 留空降级（安全） |
| ④ | L264 | `PrecipitationProbability[i]` / `[i+1]` | **无** | 长度 < 24 | **panic** |

其中风险 ④（降水概率无校验）是最容易被触发的崩溃路径——温度长度恰好 24 但降水概率缺失或长度不足时，代码直接越界。

---

### 4.3 24 小时 → 12 列聚合

[widget-weather.go#L250-L284](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L250-L284)

API 返回 24 个整点温度（`Hourly.Temperature`）和 24 个整点降水概率（`Hourly.PrecipitationProbability`）。按每 2 小时一组聚合：

```
聚合循环：for i = 0; i < 24; i += 2  →  目标索引 = i/2
┌──────────────────┬──────────────────────────────────────────────────┐
│ 字段             │ 聚合规则                                         │
├──────────────────┼──────────────────────────────────────────────────┤
│ 温度 Temperature │ ① 若该时段包含当前小时（i/2 == currentBar）       │
│                  │   → 直接使用 Current.Temperature（实时数据）     │
│                  │ ② 否则 → 两小时温度取算术平均后四舍五入          │
│                  │   → int(math.Round((t[i] + t[i+1]) / 2))         │
├──────────────────┼──────────────────────────────────────────────────┤
│ 降水 Precip      │ 两小时降水概率平均值 > 75% 时标记为 true          │
│                  │   → (p[i] + p[i+1]) / 2 > 75                    │
└──────────────────┴──────────────────────────────────────────────────┘
```

### 4.4 温度归一化与柱高映射（Scale → CSS）

[widget-weather.go#L267-L283](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L267-L283)
[weather.html#L18](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/templates/weather.html#L18)
[widget-weather.css#L42-L52](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/static/css/widget-weather.css#L42-L52)

为使柱状图在视觉上有高低差异，对 12 个温度做 Min-Max 归一化生成 `Scale`，然后经过 Go 模板 → CSS 变量 → calc() 运算，最终映射为像素高度。整个链路分三步：

**第 1 步：Min-Max 归一化生成 Scale（Go 代码）**

```
minT = slices.Min(temperatures)
maxT = slices.Max(temperatures)
range = maxT - minT

若 range > 0:
    Scale[i] = (temperatures[i] - minT) / range   ∈ [0.0, 1.0]
若 range = 0（全天恒温）:
    Scale[i] = 1.0  （全部满柱）
```

**第 2 步：模板将 Scale 注入 CSS 变量（HTML 模板）**

```html
<div class="weather-bar" style='--weather-bar-height: {{ printf "%.2f" $column.Scale }}'></div>
```

Go 的 `printf "%.2f"` 将浮点数格式化为保留两位小数的字符串，例如 `0.00`、`0.33`、`1.00`，写入每个柱子的内联样式 `--weather-bar-height`。

**第 3 步：CSS calc() 将比例转换为像素高度（样式表）**

```css
.weather-bar {
    height: calc(20px + var(--weather-bar-height) * 40px);
    width: 6px;
    mask-image: linear-gradient(0deg, transparent 0, #000 10px);
}
```

最终柱高公式：

```
height = 20px + (Scale × 40px)

Scale = 0.00  →  20px + 0px   = 20px  （最低柱，仅显示底座）
Scale = 0.50  →  20px + 20px  = 40px  （中等高度）
Scale = 1.00  →  20px + 40px  = 60px  （最高柱）
```

柱高变化范围为 **20px ~ 60px**，共 40px 的动态区间。底部 10px 通过 `mask-image` 渐变为透明，使柱子视觉上从底部柔和过渡。当前列或 hover 时，柱宽由 6px 加粗到 10px，颜色也更亮。

---

### 4.5 关键列索引汇总

| 字段            | 含义                                      | 计算方式                              |
|-----------------|-------------------------------------------|---------------------------------------|
| `CurrentColumn` | 当前时刻所在列（高亮显示）                | `now.Hour() / 2`                      |
| `SunriseColumn` | 日出所在列（日出光线起始柱）              | `sunriseHour / 2`                     |
| `SunsetColumn`  | 日落所在列（日落光线终止柱，前移 1 小时） | `(sunsetHour - 1) / 2`，下限为 0      |

### 4.6 模板渲染逻辑

[weather.html#L8-L22](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/templates/weather.html#L8-L22)

遍历 12 列时，模板叠加三个视觉层：

1. **降水指示**：`HasPrecipitation == true` 时显示雨滴图标
2. **日照条**：列索引 ∈ `[SunriseColumn, SunsetColumn]` 时显示日光色条；两端点额外附加 `sunrise` / `sunset` 渐变样式
3. **当前列高亮**：列索引 == `CurrentColumn` 时添加 `weather-column-current` 样式

温度值显示时取绝对值（`absInt`），负数通过 `weather-column-value-negative` 类另加样式区分。

---

## 五、完整执行流程与风险点

```
initialize()
    │
    ├─ 校验 location 必填
    ├─ 设置时间标签（12h/24h）
    └─ 设置单位（默认 metric）

update(ctx)
    │
    ├─ [第1次] fetchOpenMeteoPlaceFromName(location)
    │     ├─ parsePlaceName() → 拆分查询词 + 行政区过滤词
    │     ├─ 调用 Geocoding API，count=20
    │     ├─ 按行政区过滤或取首条结果
    │     └─ time.LoadLocation() → place.location
    │
    └─ fetchWeatherForOpenMeteoPlace(place, units)
          ├─ 根据 units 选定 celsius / fahrenheit
          ├─ 构造 Forecast API 请求（timezone=place.Timezone）
          ├─ now = time.Now().In(place.location) ← 时区对齐
          │
          ├─ ☣ 风险①：Daily.Sunrise[0] 空数组 panic（无守卫）
          ├─ ☣ 风险②：Daily.Sunset[0]  空数组 panic（无守卫）
          ├─ 计算 CurrentColumn / SunriseColumn / SunsetColumn
          │
          ├─ 长度守卫：len(Hourly.Temperature) == 24？
          │     ├─ 否 → 跳过聚合，bars 为空 → 预报列留空（静默降级 ✔）
          │     └─ 是 → 进入聚合循环
          │           ├─ ☣ 风险③：PrecipitationProbability[i]/[i+1]
          │           │            长度 < 24 时越界 panic（无守卫）
          │           ├─ 24h → 12 列聚合（温度平均 + 降水阈值判断）
          │           └─ Min-Max 归一化 → Scale（Go）
          │
          ├─ 模板注入 CSS 变量 --weather-bar-height
          └─ CSS calc() 映射为 20px~60px 柱高
```

### 崩溃与降级行为总结

| 异常场景 | 是否 panic | 可见表现 |
|---------|-----------|---------|
| `len(Temperature) != 24` | 否 | 预报列空白，天气文字/体感正常（静默降级） |
| `len(Sunrise) == 0` 或 `len(Sunset) == 0` | **是** | **整个进程崩溃退出**（详见第六章） |
| `len(Temperature) == 24` 且 `len(Precip) < 24` | **是** | **整个进程崩溃退出**（详见第六章） |
| API 请求整体失败（网络错误） | 否 | `canContinueUpdateAfterHandlingErr` 控制，显示错误状态 |

---

## 六、Goroutine 调度、Panic 传播与 Recover 缺口

### 6.1 Widget 更新的 Goroutine 调度链

天气 Widget 的 `update()` 并非在 HTTP 请求 goroutine 中直接执行，而是经过两层 goroutine 调度：

**第 1 层：页面级并发调度**

[glance.go#L233-L270](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/glance.go#L233-L270)

每当浏览器请求页面内容（`/api/pages/{page}/content`）时，`handlePageContentRequest` 会调用 `page.updateOutdatedWidgets()`，该函数遍历所有 Head Widgets 和 Column Widgets，为每一个需要更新的 Widget 启动独立 goroutine：

```go
func (p *page) updateOutdatedWidgets() {
    now := time.Now()
    var wg sync.WaitGroup
    ctx := context.Background()

    for w := range p.HeadWidgets {
        widget := p.HeadWidgets[w]
        if !widget.requiresUpdate(&now) { continue }
        wg.Add(1)
        go func() {
            defer wg.Done()        // ← 仅注册 Done，无 recover
            widget.update(ctx)     // ← weatherWidget.update() 在此执行
        }()
    }
    // Columns 中的 Widgets 同样模式启动 goroutine
    wg.Wait()
}
```

**第 2 层：容器级嵌套调度（可选）**

[widget-container.go#L23-L42](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-container.go#L23-L42)

若天气 Widget 被放在 Group 或 Split-Column 容器内，则容器的 `_update()` 方法会为每个子 Widget 再启动一层独立 goroutine，结构与第 1 层完全相同——同样只有 `defer wg.Done()`，没有 recover。

**调用链全景**：

```
HTTP 请求 goroutine
    └─ handlePageContentRequest
         ├─ page.mu.Lock()
         └─ page.updateOutdatedWidgets()
              ├─ goroutine A: widget.update()  ← 天气 Widget（顶层）
              │    └─ weatherWidget.update()
              │         └─ fetchWeatherForOpenMeteoPlace()
              │              └─ ☣ 数组越界 panic
              ├─ goroutine B: widget.update()  ← 其他 Widget
              ├─ goroutine C: containerWidget.update()
              │    └─ containerWidgetBase._update()
              │         ├─ goroutine C1: subWidget.update()
              │         └─ goroutine C2: weatherWidget.update()  ← 嵌套的天气 Widget
              │              └─ ☣ 数组越界 panic
              └─ wg.Wait()
```

### 6.2 Panic 传播路径与影响范围

在 Go 语言中，**panic 仅在当前 goroutine 内传播，不能跨 goroutine 被父 goroutine 的 recover 捕获**。结合代码分析传播路径：

```
goroutine A（天气 Widget 更新）
    │
    ├─ weatherWidget.update(ctx)
    │    └─ fetchWeatherForOpenMeteoPlace()
    │         ├─ ☣ p[i] 越界 / Sunrise[0] 越界
    │         │
    │         └─ panic("index out of range")
    │              │
    │              ▼
    │         栈展开，执行 defer wg.Done()  ✔ WaitGroup 会计数减 1
    │              │
    │              ▼
    │         无 recover → goroutine A 终止
    │              │
    │              ▼
    │         Go runtime 检测到未恢复 panic → **整个进程终止**
    │
    ├─ goroutine B（其他 Widget） → 随进程一同被杀死
    ├─ goroutine C（容器）       → 随进程一同被杀死
    └─ wg.Wait()                 → 永远不会返回（进程已死）
```

**关键影响点**：

| 影响对象 | 行为 |
|---------|------|
| WaitGroup | `defer wg.Done()` 在 panic 展开时仍会执行，计数会减 1，不会永久阻塞 |
| page.mu 互斥锁 | 锁在 HTTP 请求 goroutine 中（不在子 goroutine 内），进程死亡后锁自然消失，无死锁 |
| 其他 Widget | 同一次请求中正在更新的其他 goroutine 全部被 runtime 杀死，可能部分更新到一半 |
| HTTP 响应 | `handlePageContentRequest` 所在 goroutine 也随进程终止，客户端收到连接被重置 |
| 后续请求 | 进程退出，服务停止，所有用户均无法访问 |
| 下次"重试" | **不存在重试**。进程已死，需由外部进程管理器（systemd / Docker）重启 |

之前对"整个 Widget 更新失败，下次重试"的理解是**错误的**——数组越界 panic 会导致 Glance **整个进程崩溃退出**，而不是仅单个 Widget 失败。

### 6.3 Recover 缺口盘点

对整个代码库搜索 `recover` 关键字，仅在一处发现相关注释：

[widget-custom-api.go#L223](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-custom-api.go#L223) 中提到 `// handles recovering from panics`，但这是 Custom API Widget 内部执行用户自定义脚本时使用的局部 recover，不覆盖 Widget update 入口。

**三层 goroutine 启动位置均无 recover**：

| 位置 | 代码 | 是否有 recover |
|------|------|--------------|
| `page.updateOutdatedWidgets()` [glance.go#L247-L250](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/glance.go#L247-L250) | `go func() { defer wg.Done(); widget.update(ctx) }()` | ❌ 无 |
| `page.updateOutdatedWidgets()` [glance.go#L262-L265](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/glance.go#L262-L265) | （Column Widgets 同一模式） | ❌ 无 |
| `containerWidgetBase._update()` [widget-container.go#L35-L38](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-container.go#L35-L38) | `go func() { defer wg.Done(); widget.update(ctx) }()` | ❌ 无 |

同时，`main()` → `glance.Main()` → `serveApp()` 的启动链路上也没有全局 panic 恢复机制。

### 6.4 从数组越界到进程死亡的完整时间线

```
T+0ms   用户浏览器请求 /api/pages/home/content
T+1ms   handlePageContentRequest 获取 page.mu 锁，调用 updateOutdatedWidgets()
T+2ms   天气 Widget 需要更新，启动 goroutine A
T+3ms   goroutine A 执行 fetchWeatherForOpenMeteoPlace()
        ↓
        API 返回 len(Temperature)=24, len(PrecipitationProbability)=0
        ↓
        进入 if len(Temperature)==24 分支
        ↓
        循环 i=0: p[0] → 越界
        ↓
T+4ms   panic: runtime error: index out of range [0] with length 0
        ↓
        defer wg.Done() 执行 → WaitGroup 计数 -1
        ↓
        无 recover → Go runtime 开始终止所有 goroutine
T+5ms   整个进程以非零退出码退出
        ↓
        用户浏览器显示连接被重置（ECONNRESET）
        ↓
        其他正在浏览的用户也全部断开
        ↓
        systemd / Docker 根据重启策略决定是否拉起新进程
```
