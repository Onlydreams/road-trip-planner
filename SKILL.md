---
name: road-trip-planner
description: >
  基于高德地图 personal-map skill 规划多日自驾游路线，生成连续路线个人地图小程序二维码，并输出每日行程详情。当用户提到自驾、公路旅行、路线规划、旅游行程、生成地图、规划环线、控制驾驶时间或查询沿途里程时使用。支持闭合环线设计和跨天里程补全。
  (China only — requires Amap API Key and @lbs-amap/personal-map skill)
  Plan multi-day road trip routes using Amap APIs. Generate continuous-route personal map QR codes and daily itinerary details. Triggered when users mention road trips, route planning, travel itineraries, map generation, loop routes, driving time control, or mileage queries. Supports closed-loop design and cross-day mileage completion.
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
| `lbs-amap/personal-map` | [ClawHub](https://clawhub.ai/lbs-amap/personal-map) | Amap Web Service API wrapper / 高德 Web 服务 API 封装 |
| Amap API Key | [Amap Open Platform](https://lbs.amap.com/) | Required env var: `AMAP_API_KEY` / 需配置环境变量 |

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

Uses the `AMapPersonalMapClient` from the `personal-map` skill:

```python
from scripts.amap_personal_map_client import AMapPersonalMapClient
client = AMapPersonalMapClient(api_key=os.getenv("AMAP_API_KEY"))
```

For each location:
1. `maps_geo(address)` — get precise coordinates
2. `maps_text_search(name, offset=5)` — get poiId
3. Verify search result coordinates are within 30km of geo coordinates to prevent cross-city mismatches
4. `time.sleep(0.3)` between API calls to avoid QPS throttling

对每个地点执行：`maps_geo` 获取坐标 → `maps_text_search` 获取 poiId → 验证距离 < 30km → sleep 0.3s。

### Step 4: Driving route planning with cross-day completion / 驾车路径规划（跨天补全）

When calculating daily mileage, **must** insert the previous day's endpoint as the start of the current day's points list:

```python
for i, day in enumerate(days):
    points = [p for p in day["points"] if "lon" in p]
    if i > 0:
        prev = [p for p in days[i-1]["points"] if "lon" in p]
        if prev:
            points.insert(0, prev[-1])
    # Call maps_direction_driving segment by segment
```

计算每日里程时，**必须**把前一天的终点插入当天 points 列表头部作为起点。

### Step 5: Generate personal map QR code / 生成个人地图二维码

1. Collect all valid points (poiId not empty and not placeholder)
2. Deduplicate by name, but **preserve the start/end city if it appears at both ends**
3. Trim to ≤ 16 nodes per route
4. Call `maps_schema_personal_map` with `sceneType=3` (route planning mode)
5. Download QR code image and display

收集有效点 → 按名称去重（保留首尾同名终点）→ 精简至 ≤ 16 节点 → 调用 `maps_schema_personal_map`(sceneType=3) → 下载二维码。

### Step 6: Output Markdown itinerary / 输出 Markdown 行程表

Output directly in the conversation as Markdown, including:
- Itinerary overview table (day, route, mileage, duration, attractions)
- Daily detailed itinerary (waypoints, coordinates, notes)
- Practical tips (altitude sickness, tickets, road conditions, temperature differences, etc.)
- QR code image and fallback link

在对话中直接输出 Markdown：行程概览表格、每日详细行程、实用提示、二维码图片和备用链接。

## Key Constraints / 关键约束

| Constraint / 约束 | Description / 说明 |
|-------------------|-------------------|
| Max nodes per route / 单条路线节点上限 | Amap `lineList` max 16 nodes per `pointInfoList` |
| poiId required / poiId 必填 | `maps_schema_personal_map` requires non-empty poiId |
| poiId strategy / poiId 获取策略 | `maps_text_search` first → retry with city name → fallback placeholder `B000000000` |
| QPS limit / QPS 限制 | ≥ 0.3s interval between calls, or `CUQPS_HAS_EXCEEDED_THE_LIMIT` |
| API Key / API Key | Must set `AMAP_API_KEY` env var or pass to constructor |
| Region / 地域限制 | China only — Amap (高德) covers mainland China |

## Output Template / 输出模板

```markdown
# [Route Name] Road Trip Guide / 自驾行程指南

> Start/End: [City]　> Total: ~X km　> Days: X

## Itinerary Overview / 行程概览

| Day | Route | Est. Mileage | Est. Duration | Key Attractions |
|-----|-------|-------------|--------------|-----------------|
| Day 1 | ... | ... | ... | ... |

## Daily Itinerary / 每日详细行程

### Day 1: [Title]
- **Driving mileage**: X km
- **Driving duration**: ~X hours

**Waypoints / 途经点**：
- **Name**: Note `coordinates`

## Practical Tips / 实用提示
1. ...

## Personal Map QR Code / 个人地图二维码
![Route Name](file:///...)
```

## Advanced Capabilities / 进阶能力

- **Supply point search / 沿途补给搜索**：Use `maps_around_search` near key nodes (e.g., towns before uninhabited areas) to find gas stations, hospitals, charging stations
- **Mileage anomaly troubleshooting / 里程异常排查**：If a day's mileage is 0 or obviously wrong, check whether the previous day's endpoint was correctly inserted, and whether coordinates are duplicated
- **Seasonal adjustments / 季节性调整**：Peak season (Jul–Aug) requires advance hotel booking; some winter road sections (e.g., Qilian Mountains) may be closed, requiring detours

## References / 参考

- Full examples: [examples.md](examples.md)
- Underlying API: [@lbs-amap/personal-map](https://clawhub.ai/lbs-amap/personal-map) skill
