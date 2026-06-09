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

### 4.2 24 小时 → 12 列聚合

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

### 4.3 温度归一化（Scale 计算）

[widget-weather.go#L267-L283](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/widget-weather.go#L267-L283)

为使柱状图在视觉上有高低差异，对 12 个温度做 Min-Max 归一化：

```
minT = slices.Min(temperatures)
maxT = slices.Max(temperatures)
range = maxT - minT

若 range > 0:
    Scale[i] = (temperatures[i] - minT) / range   ∈ [0, 1]
若 range = 0（恒温）:
    Scale[i] = 1  （全部满柱）
```

该 `Scale` 值通过 CSS 变量 `--weather-bar-height` 控制柱状条高度百分比。

### 4.4 关键列索引汇总

| 字段            | 含义                                      | 计算方式                              |
|-----------------|-------------------------------------------|---------------------------------------|
| `CurrentColumn` | 当前时刻所在列（高亮显示）                | `now.Hour() / 2`                      |
| `SunriseColumn` | 日出所在列（日出光线起始柱）              | `sunriseHour / 2`                     |
| `SunsetColumn`  | 日落所在列（日落光线终止柱，前移 1 小时） | `(sunsetHour - 1) / 2`，下限为 0      |

### 4.5 模板渲染逻辑

[weather.html#L8-L22](file:///d:/fz/0601/solo-dogfeeding/code/138-glance/internal/glance/templates/weather.html#L8-L22)

遍历 12 列时，模板叠加三个视觉层：

1. **降水指示**：`HasPrecipitation == true` 时显示雨滴图标
2. **日照条**：列索引 ∈ `[SunriseColumn, SunsetColumn]` 时显示日光色条；两端点额外附加 `sunrise` / `sunset` 渐变样式
3. **当前列高亮**：列索引 == `CurrentColumn` 时添加 `weather-column-current` 样式

温度值显示时取绝对值（`absInt`），负数通过 `weather-column-value-negative` 类另加样式区分。

---

## 五、完整执行流程

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
          ├─ 计算 CurrentColumn / SunriseColumn / SunsetColumn
          ├─ 24h → 12 列聚合（温度平均 + 降水阈值判断）
          ├─ Min-Max 归一化 → Scale
          └─ 返回 weather{}
```
