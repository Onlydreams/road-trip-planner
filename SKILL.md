---
name: road-trip-planner
description: >
  基于高德地图 personal-map skill 规划多日自驾游路线，生成连续路线个人地图小程序二维码，并输出每日行程详情。当用户提到自驾、公路旅行、路线规划、旅游行程、生成地图、规划环线、控制驾驶时间或查询沿途里程时使用。支持闭合环线设计和跨天里程补全。
  Plan multi-day road trip routes using Amap APIs. Generate continuous-route personal map QR codes and daily itinerary details. Triggered when users mention road trips, route planning, travel itineraries, map generation, loop routes, driving time control, or mileage queries. Supports closed-loop design and cross-day mileage completion. (China only — requires Amap API Key and @lbs-amap/personal-map skill)
depends: lbs-amap/personal-map
region: China only
---

# Road Trip Planner / 自驾游路线规划

> China only — 本 skill 基于高德地图 API，仅适用于中国大陆地区。  
> Dependency: [@lbs-amap/personal-map](https://clawhub.ai/lbs-amap/personal-map) (ClawHub) / [GitHub](https://github.com/amap-demo/amap-sdk-skills)

## Trigger Conditions / 触发条件

Trigger when the user has any of these intents:

当用户出现以下任意意图时，主动调用本 skill：

- Plan a road trip / road trip route / loop route (规划自驾游路线 / 公路旅行 / 环线)
- Design a travel itinerary involving multi-city driving (设计涉及多地驾驶的旅游行程)
- Generate a personal map / travel map / route QR code (生成个人地图 / 旅游地图 / 路线二维码)
- Control daily driving time / avoid fatigued driving (控制每日驾驶时间 / 避免疲劳驾驶)
- Query mileage, gas stations, hospitals along the route (查询沿途里程、加油站、医院等补给点)

## Dependencies / 前置依赖

| Dependency | Source | Description |
|-----------|--------|-------------|
| `lbs-amap/personal-map` | [ClawHub](https://clawhub.ai/lbs-amap/personal-map) | Amap Web Service API wrapper / 高德 Web 服务 API 封装。安装命令：`openclaw skills install personal-map`。提供 `AMapPersonalMapClient` 类（import 路径 `scripts.amap_personal_map_client`），本仓库自身不含该脚本。 |
| Amap API Key | [Amap Open Platform](https://lbs.amap.com/) | Required env var: `AMAP_API_KEY` / 需配置环境变量 |

### Dependency Availability Check / 依赖可用性检查

Before any API call, verify that the `personal-map` skill is installed and the API key is configured. If either is missing, stop and tell the user what's needed — do not attempt to proceed.

在任何 API 调用之前，先检查 `personal-map` skill 是否已安装以及 API Key 是否已配置。任一缺失则停止并告知用户缺失项，不要尝试继续。

```
Checklist / 检查清单:
1. personal-map skill installed? → No → 提示: "请先安装 personal-map skill，运行: openclaw skills install personal-map"
2. AMAP_API_KEY set? → No → 提示: "请先配置高德 API Key: export AMAP_API_KEY=your_key，获取地址 https://lbs.amap.com/"
3. AMapPersonalMapClient importable? → No → 提示: "personal-map skill 版本可能不兼容，请确认其提供 AMapPersonalMapClient 类"
```

## Core Principles / 核心原则

1. **No fatigued driving / 不疲劳驾驶**：Keep daily actual driving time under 6 hours, ideally 4–5 hours. For high-altitude, mountainous, or winding roads, further reduce driving time.（每日实际驾驶时间控制在 6 小时以内，理想状态 4-5 小时。高原、山区、弯道多的路段进一步压缩。）

2. **Continuous route / 连续路线**：Merge all daily waypoints into one continuous route, either a closed loop or with a clear endpoint.（所有天的节点合并为一条连续路线，首尾闭合或明确终点。）

3. **Closed loop / 闭合环线**：If start and end are the same city, ensure the endpoint is preserved after deduplication.（如果起点和终点是同一城市，确保终点节点在去重后仍然保留在路线末尾。）

4. **Daily flexibility / 每日弹性**：Leave buffer room in the itinerary; avoid over-scheduling.（行程中保留机动空间，避免把时间排得太满。）

## Workflow / 工作流程

### Step 1: Confirm itinerary framework / 确认行程框架

Confirm with the user:
- Start point, end point (loop or not)
- Total number of days
- Must-visit attractions / cities
- Travel season (affects road conditions and accommodation)

与用户确认以下信息：起点、终点（是否环线）、总天数、必去景点/途经城市、出发季节。

If unspecified, proactively suggest reasonable defaults.

### Step 2: Design daily waypoints / 设计每日节点

Split waypoints by day following these rules:
- 4–6 hours of driving per day, never exceed 6 hours
- Each day's `points` list includes the core locations passed that day
- The end city of each day must also serve as the start point for the next day (for mileage calculation)

按天拆分节点：每天驾驶 4-6 小时，不超过 6 小时；每天的 points 列表包含当天途经的核心地点；终点城市需同时作为下一天的起点。

### Step 3: Call Amap APIs / 调用高德 API

Uses `AMapPersonalMapClient` provided by the `personal-map` skill. The class is at `scripts.amap_personal_map_client` inside that skill's directory — this is the canonical import path. This repo itself contains no scripts.

```python
import os
from scripts.amap_personal_map_client import AMapPersonalMapClient

# Check API key BEFORE creating client
api_key = os.getenv("AMAP_API_KEY")
if not api_key:
    print("AMAP_API_KEY not set. Get a key at https://lbs.amap.com/ and run:")
    print("  export AMAP_API_KEY=your_key")
    # stop here — do not proceed

# Create client (auto-reads AMAP_API_KEY from env):
client = AMapPersonalMapClient()
# Or pass explicitly: client = AMapPersonalMapClient(api_key="your_key")
```

**IMPORTANT**: All API methods return structured dicts on error, they do NOT throw exceptions. Always check for `"error"` key:

```python
result = client.maps_geo("北京市朝阳区", "北京")
if isinstance(result, dict) and "error" in result:
    print(f"API error: {result['message']}")
    # handle error — do not assume result contains coordinates
```

For each location:
1. `maps_geo(address, city)` — get precise coordinates. Pass the city name as second arg for better accuracy.
2. `maps_text_search(keywords, city, offset=5)` — get poiId. Use `offset=5` to limit results (API default is 20).
3. Verify search result coordinates are within 30km of geo coordinates to prevent cross-city mismatches
4. `time.sleep(0.3)` between API calls to avoid QPS throttling

对每个地点执行：`maps_geo` 获取坐标 → `maps_text_search` 获取 poiId → 验证距离 < 30km → sleep 0.3s。

**Error handling / 错误处理**：
- If `AMapPersonalMapClient` is not found → check the pre-flight checklist in the Dependencies section, inform user, and **stop**
- If `AMAP_API_KEY` is empty → inform user and **stop**
- If API returns `{"error": ...}` → check `result["message"]`, skip this location with a warning, use the previous valid result as fallback
- If `maps_geo` returns error → skip this location with a warning
- If `maps_text_search` returns error or no result within 30km → retry with `{city_name} {location_name}`, if still empty use placeholder `B000000000`
- If QPS exceeded (`CUQPS_HAS_EXCEEDED_THE_LIMIT` in error message) → wait 1s and retry once

#### 3.1 Supply point search per day / 每日补给搜索

After all waypoint coordinates are resolved, search for supply points near key nodes on **days that enter sparsely populated areas or have long uninterrupted stretches**:

对于进入人烟稀少区域或存在长距离无补给路段的日期，在关键节点附近搜索补给点：

```python
# For each day flagging a supply concern (long stretch, remote area):
# Search gas stations and hospitals near the day's midpoint or last town before uninhabited area
result = client.maps_around_search(
    keywords="加油站|医院",
    location=f"{lon},{lat}",    # "经度,纬度" 格式
    radius=5000,               # 搜索半径 5km（默认 1000m）
    offset=5
)
if isinstance(result, dict) and "error" in result:
    print(f"补给搜索失败: {result['message']}")
```

**When to search / 何时搜索**：
| Condition / 条件 | Action / 动作 |
|------------------|---------------|
| Day mileage > 300km / 日里程超 300km | Search gas stations at midpoint / 在中点搜索加油站 |
| Entering uninhabited area (>100km no town) / 进入无人区（>100km 无城镇） | Search all supply points at the last town before entry / 在进入前最后城镇搜索全部补给点 |
| High-altitude segment (>3500m) / 高海拔路段（>3500m） | Search hospitals at the nearest low-altitude town / 在最近的低海拔城镇搜索医院 |
| All days / 每天 | Search at least one hospital near the day's endpoint (overnight city) / 在每天终点城市至少搜索一家医院 |

### Step 4: Driving route planning with cross-day completion / 驾车路径规划（跨天补全）

When calculating daily mileage, **must** insert the previous day's endpoint as the start of the current day's points list:

```python
for i, day in enumerate(days):
    points = [p for p in day["points"] if "lon" in p]
    if i > 0:
        prev = [p for p in days[i-1]["points"] if "lon" in p]
        if prev:
            points.insert(0, prev[-1])
    # Drive segment by segment using maps_direction_driving:
    #   For each adjacent pair in points, call:
    #     result = client.maps_direction_driving(
    #         origin=f"{points[j]['lon']},{points[j]['lat']}",
    #         destination=f"{points[j+1]['lon']},{points[j+1]['lat']}"
    #     )
    #   Sum distances for the day's total.
    #   Parameter format: "longitude,latitude" (no spaces).
```

计算每日里程时，**必须**把前一天的终点插入当天 points 列表头部作为起点。
`maps_direction_driving` 的 `origin` / `destination` 参数格式为 `"经度,纬度"`（逗号分隔，无空格）。

### Step 5: Generate personal map QR code / 生成个人地图二维码

#### 5.1 Collect valid points / 收集有效点

Collect all points across all days that have a non-empty poiId (skip placeholder `B000000000`). Preserve insertion order (day-by-day).

收集所有天的有效点（poiId 非空且非占位符 `B000000000`），按天顺序保持插入顺序。

#### 5.2 Deduplicate by name / 按名称去重

Rules (apply in order):

1. **Closed-loop anchor / 环线锚点**：If the first and last points have the same name AND this name also appears in the middle, keep the first AND last occurrence, remove the middle one(s). This ensures the route stays closed.
2. **Intermediate duplicates / 中间重复**：For all other duplicate names, keep the **first** occurrence, remove subsequent ones.
3. **Adjacent duplicates / 连续重复**：If two consecutive points share the same name, remove one (keep the earlier day's context). This happens when a day's endpoint equals the next day's start point due to cross-day completion.

```
Deduplication flow:
  names = [p["name"] for p in points]
  if names[0] == names[-1]:
      → keep points[0] and points[-1], remove middle occurrences of this name
  for all other duplicates:
      → keep first occurrence, remove rest
```

#### 5.3 Trim to ≤ 16 nodes / 精简至 ≤ 16 节点

If the deduplicated list exceeds 16 nodes, trim using this priority order:

| Priority | Keep | Drop |
|----------|------|------|
| 1 (highest) | Start and end points (环路锚点) | — |
| 2 | User-specified must-visit attractions | — |
| 3 | Major scenic spots / landmarks (景点/地标) | Rest stops, service areas (服务区) |
| 4 | Geographically isolated nodes (唯一覆盖该区域的点) | Dense urban clusters (keep 1 per city) |
| 5 | POI with non-placeholder poiId | POI with placeholder poiId `B000000000` |

**Geographic distribution principle / 地理分布原则**：When choosing between nodes of equal priority, prefer nodes that are farther apart (ensure the route visually represents the full journey). Avoid dropping the only node in a sparse region.

**Minimum trim / 最小精简**：Only remove as many nodes as needed to reach 16. Never remove the start or end anchor.

#### 5.4 Data integrity before calling API / 调用 API 前的数据完整性

```python
# Build lineList structure
point_info_list = [
    {"name": p["name"], "lon": p["lon"], "lat": p["lat"], "poiId": p["poiId"]}
    for p in deduplicated_points
]
line_list = [{
    "title": "Route Name / 路线名称",
    "pointInfoList": point_info_list
}]

# Validate before calling maps_schema_personal_map
if len(point_info_list) > 16:
    raise ValueError(f"Still {len(point_info_list)} nodes, must be ≤ 16")
if any(p["poiId"] == "B000000000" or not p["poiId"] for p in point_info_list):
    raise ValueError("Placeholder or empty poiId found")
```

#### 5.5 Call API and download QR / 调用 API 并下载二维码

```python
result = client.maps_schema_personal_map(
    orgName="路线名称",
    lineList=line_list,
    sceneType=3          # 3 = route planning mode (仅创建路线)
)

# Check for API errors (all methods return dict on error, no exceptions)
if isinstance(result, dict) and "error" in result:
    print(f"地图生成失败: {result['message']}")
    # fallback: output the coordinates as text so user can manually create route
else:
    qr_url = result["qr_code_url"]
    # Download QR image:
    import urllib.request
    qr_path = "/tmp/road_trip_qr.png"
    urllib.request.urlretrieve(qr_url, qr_path)
    # Display inline: ![路线地图](file:///tmp/road_trip_qr.png)
    # Fallback link: [在 Amap 中打开](qr_url)
```

1. Call `maps_schema_personal_map` with `sceneType=3` (route planning mode / 仅创建路线)
2. Check for error in result dict (no exceptions thrown)
3. Download QR code image and display inline
4. Always include the fallback URL link

收集有效点 → 按名称去重（按上述三条规则）→ 构建 lineList → 按优先级精简至 ≤ 16 节点 → 数据完整性校验 → 调用 `maps_schema_personal_map`(sceneType=3) → 检查错误 → 下载二维码 → 显示备用链接。

### Step 6: Output Markdown itinerary / 输出 Markdown 行程表

Output directly in the conversation as Markdown, including:
- Itinerary overview table (day, route, mileage, duration, road type, attractions) with total row
- Daily detailed itinerary (waypoints table with coordinates and altitude, supply points)
- Practical tips (altitude sickness, road conditions, tickets, temperature, seasonal notes)
- QR code image and fallback link

在对话中直接输出 Markdown：行程概览表格（含总里程合计行）、每日详细行程（含海拔和补给点）、实用提示、二维码图片和备用链接。

## Key Constraints / 关键约束

| Constraint / 约束 | Description / 说明 |
|-------------------|-------------------|
| Dependency check / 依赖检查 | Before any API call, verify `personal-map` skill is installed AND `AMAP_API_KEY` is set. Fail fast with clear user-facing messages if missing. |
| Max nodes per route / 单条路线节点上限 | Amap `lineList` max 16 nodes per `pointInfoList` |
| poiId required / poiId 必填 | `maps_schema_personal_map` requires non-empty poiId |
| poiId strategy / poiId 获取策略 | `maps_text_search` first → retry with city name → fallback placeholder `B000000000` |
| API error pattern / API 错误模式 | All API methods return `{"error": ..., "message": ...}` on failure — they do NOT throw exceptions. Always check for `"error"` key in every response. |
| QPS limit / QPS 限制 | ≥ 0.3s interval between calls, or `CUQPS_HAS_EXCEEDED_THE_LIMIT` |
| API Key / API Key | Set `AMAP_API_KEY` env var (auto-read by `AMapPersonalMapClient()` with no args) or pass explicitly |
| Region / 地域限制 | China only — Amap (高德) covers mainland China |
| External dependency / 外部依赖 | This repo contains no scripts. `AMapPersonalMapClient` lives in the `personal-map` skill at `scripts.amap_personal_map_client`. Install via `openclaw skills install personal-map`. |

## Output Template / 输出模板

```markdown
# [Route Name] Road Trip Guide / 自驾行程指南

> Start/End: [City]　> Total: ~X km　> Days: X　> Max Altitude: ~Xm

## Itinerary Overview / 行程概览

| Day | Route | Est. Mileage | Est. Duration | Road Type | Key Attractions |
|-----|-------|-------------|--------------|-----------|-----------------|
| Day 1 | ... | ... | ... | 高速/国道/山路 | ... |
| **Total** | — | **X km** | **~X h** | — | — |

## Daily Itinerary / 每日详细行程

### Day 1: [Title]
- **Driving mileage**: X km (高速 Xkm / 国道 Xkm / 山路 Xkm)
- **Driving duration**: ~X hours
- **Max altitude**: ~Xm (仅高原线路标注 / only for high-altitude routes)

**Waypoints / 途经点**：
| # | Name | Coordinates | Altitude | Note |
|---|------|------------|----------|------|
| 1 | ... | (lat, lng) | ~Xm | ... |

**Supply points / 沿途补给**：
- ⛽ Gas stations / 加油站: [name] at [location]
- 🏥 Hospitals / 医院: [name] at [location]

## Practical Tips / 实用提示
1. **Altitude sickness / 高原反应**：...（高海拔线路必须包含 / required for routes >3000m）
2. **Road conditions / 路况**：...
3. **Tickets & booking / 门票与预订**：...
4. **Temperature & clothing / 温差与衣物**：...

## Personal Map QR Code / 个人地图二维码
![Route Name](file:///...)
> Fallback link / 备用链接: [Open in Amap](url)
```

## Advanced Capabilities / 进阶能力

- **Supply point search / 沿途补给搜索**：Integrated into Step 3.1 — conditions and API calls are defined there. / 已集成到 Step 3.1，按条件触发。
- **Mileage anomaly troubleshooting / 里程异常排查**：If a day's mileage is 0 or obviously wrong, check whether the previous day's endpoint was correctly inserted (Step 4 cross-day completion), and whether coordinates are duplicated.
- **Seasonal adjustments / 季节性调整**：See below. / 见下方季节性检查清单。

### Seasonal Checklist / 季节性检查清单

> **通用提醒**：暑期 (7–8月) 全国旺季，住宿提前 1–2 周预订、景区拥挤。国庆 (10.1–10.7) 各热门线路住宿极端紧张、价格翻倍，部分路段拥堵严重。
> **General rule**: Summer (Jul–Aug) = nationwide peak season — book 1–2 weeks ahead. National Day (Oct 1–7) = extreme crowding and price surges on all popular routes.

#### 节假日影响 / Holiday Impact

> 农历节日（春节、清明、端午、中秋）每年公历日期不同，规划时请确认当年具体日期。
> Lunar-calendar holidays shift each year — confirm exact dates when planning.

| 节假日 Holiday | 时间 When | 天数 | 自驾影响 Road Trip Impact |
|---------------|-----------|------|--------------------------|
| **元旦** New Year | 12.31–1.2 | 3天 | 短途游小高峰，景区中等拥挤，高速不免费 |
| **寒假** Winter Break | 1–2月 | 约1月 | 学生家庭出游季，三亚/西双版纳/北海等暖冬目的地旺，东北冰雪游旺季 |
| **春节** Spring Festival | 1月底–2月中 | 7–8天 | 全国性出行最高峰之一，高速免费，热门景区爆满，大部分商家歇业至初七，除夕/初一加油站可能休息 |
| **清明** Qingming | 4月初 | 3天 | 短途踏青高峰，高速免费，城郊山路拥堵，墓地周边交通管制 |
| **五一** Labor Day | 5.1–5.5 | 5天 | 仅次于国庆的出行高峰，高速免费，热门票/住宿需提前 1 月预订，景区限流常见 |
| **端午** Dragon Boat | 5月底–6月 | 3天 | 短途游为主，高速不免费，影响相对较小；南方进入雨季注意山路 |
| **暑期** Summer Vacation | 7–8月 | 约2月 | 全国旺季，家庭出游主力，住宿全线紧张，景区排队严重；西北/高原线路集中出行 |
| **中秋** Mid-Autumn | 9月中–10月初 | 3天 | 短途探亲为主；若与国庆连休可形成 8 天+ 超长假期，出行量倍增 |
| **国庆** National Day | 10.1–10.7 | 7天 | 全年最大出行高峰，高速免费，热门线路住宿翻 3–5 倍，部分景区限流劝返，G318/独库/青甘等拥堵严重 |
| **胡杨林季** Ejina Poplar | 10月初–10月中 | 约2周 | 额济纳/酒泉/塔里木胡杨林景区周边住宿极端紧张（提前 1 月+），价格数倍上涨 |
| **林芝桃花节** Nyingchi Peach | 3月底–4月中 | 约3周 | 林芝/波密住宿紧张，G318 川藏线早春进藏车流增多 |

#### 分区域季节风险 / Seasonal Risks by Region

> 下表按区域列出四季关键风险。暑期和法定节假日叠加效应请同时参考上方节假日表。
> The table below lists key seasonal risks by region. Cross-reference with the holiday calendar above for compound effects (e.g., summer vacation + regional rainy season).

| Region / 区域 | Best / 最佳 | 冬 11–3月 | 春 3–5月 | 夏 6–8月 | 秋 9–10月 |
|---------------|-------------|-----------|----------|----------|-----------|
| **川西** Western Sichuan | 9–10月 | 垭口积雪封路，备防滑链 / passes closed by snow, carry chains | 残雪渐融，3–4月金川梨花 | G318/G317 雨季塌方泥石流 / landslides | 摄影黄金季，10月中下旬可能降雪 |
| **G318 川藏线** Sichuan-Tibet Hwy | 5月, 9–10月 | 折多山/东达山暗冰，交通管制 / black ice at high passes | 林芝桃花(3–4月)，然乌湖最美，垭口仍有残雪 | 通麦/海通沟塌方，住宿翻倍 / mudslides, lodging 2× | 最佳进藏窗口，国庆住宿极端紧张 |
| **稻城亚丁** Daocheng Yading | 9–10月 | 牛奶海结冰，景区可能限流 / iced lakes, restricted access | 4–5月杜鹃花季，但草甸未全绿 | 雨季山路泥泞，冲古寺至洛绒牛场段注意 | 最佳季节，10月中下旬降雪，国庆人流最大 |
| **西藏阿里** Tibet Ngari | 9–10月 | 极寒，非铺装路封闭，客栈歇业，加油站 200–300km | 部分路段仍封闭，不建议 / still closed | 雨季泥泞，医疗匮乏，建议备卫星电话 / satellite phone advised | 最佳但夜间骤降，全年需备副油 / carry spare fuel |
| **珠峰线** Everest Base Camp | 4–5月, 9–10月 | 夜间 -20~-30°C，但晴空视野最佳 / coldest but clearest views | 最佳窗口，旺季价高，需边防证 | 季风遮挡珠峰，进山道路雨季 / monsoon obscures peak | 国庆后人流骤减，性价比高 / fewer crowds |
| **G109 青藏线** Qinghai-Tibet Hwy | 9–10月 | 唐古拉/昆仑山口积雪，200km+ 无人区，极寒 | 冻土开始融化，路面不平 / thawing permafrost | 冻土翻浆波浪路，大货车密集，高反是#1风险 | 最佳季节，冻土回冻路面改善，国庆车流叠加 |
| **青海 / 祁连** Qinghai | 7–9月 | 青海湖结冰草原枯黄，祁连垭口封路 | 沙尘暴，青海湖 3月前仍结冰 | 油菜花(7月)，祁连草原最佳，门源花期短 | 摄影黄金季，夜间寒冷，10月中下旬可能降雪 |
| **甘南** Gannan | 6–8月 | 垭口积雪，扎尕那可能关闭 / passes closed | 草甸未青，干燥多风 / meadows still brown | 若尔盖/桑科草原最佳，雨季山路注意 | 9月秋景最佳，国庆住宿紧张 |
| **甘肃河西走廊** Hexi Corridor | 5–6月, 9–10月 | 极寒干燥，部分景区淡季维护 / extreme cold, some sites closed | 沙尘暴高发，能见度骤降 / sandstorms | 40°C+ 暴晒，莫高窟票极紧张(提前1月约) | 最佳窗口，国庆人流爆炸 |
| **新疆北疆** N. Xinjiang | 6–8月 | <-30°C 不适合普通自驾 / too cold for regular vehicles | 融雪期泥泞，不推荐 / muddy, not recommended | 伊犁草原最佳，旺季住宿紧张 | 喀纳斯/禾木秋景(9月底–10月初)，国庆人最多 |
| **独库公路** Duku Highway | 6–9月 | ⚠️ 封闭 | 5月底视除雪进度开放 | 全段开放，旺季车多 | 10月初降雪后封闭 / closes after first snow |
| **新疆沙漠公路** Desert Highways | 4–5月, 9–10月 | 寒冷但可通行 / cold but passable | 沙尘暴，能见度近零，飞沙伤车漆 | 路面 70°C+，爆胎风险，200km+ 无补给，备副油+补胎工具 | 最佳窗口，昼夜温差大 |
| **滇西北** NW Yunnan | 4–5月, 9–10月 | 白马雪山积雪结冰，丽江-香格里拉段注意 | 干燥多风，草甸未青 | 虎跳峡/梅里路段塌方，山路泥泞 | 天气稳定，国庆人流多 |
| **蒙东呼伦贝尔** Hulunbuir | 6–9月 | <-30°C，草原枯黄封路，不推荐自驾 / not recommended | 草原未绿，不建议 / still brown | 草原绿色最佳窗口，恩和/室韦/黑山头住宿紧 | 9月中起草原枯黄，国庆最后一波 |

## References / 参考

- Full examples: [examples.md](examples.md)
- Underlying API: [@lbs-amap/personal-map](https://clawhub.ai/lbs-amap/personal-map) skill
