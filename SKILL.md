---
name: road-trip-planner
description: 基于高德 personal-map skill 规划多日自驾游路线，生成连续路线个人地图小程序二维码，并输出每日行程详情。当用户提到自驾、公路旅行、路线规划、旅游行程、生成地图、规划环线、控制驾驶时间或查询沿途里程时使用。支持闭合环线设计和跨天里程补全。
---

# Road Trip Planner

## 触发条件

当用户出现以下任意意图时，主动调用本 skill：
- 规划自驾游路线 / 公路旅行 / 环线
- 设计旅游行程（涉及多地驾驶）
- 生成个人地图 / 旅游地图 / 路线二维码
- 控制每日驾驶时间 / 避免疲劳驾驶
- 查询沿途里程、加油站、医院等补给点

## 核心原则

1. **不疲劳驾驶**：每日实际驾驶时间控制在 6 小时以内，理想状态 4-5 小时。高原、山区、弯道多的路段，驾驶时间进一步压缩。
2. **连续路线**：所有天的节点合并为一条连续路线，首尾闭合（环线）或明确终点。
3. **闭合环线**：如果起点和终点是同一城市，确保终点节点在去重后仍然保留在路线末尾。
4. **每日弹性**：行程中保留机动空间，避免把时间排得太满。

## 工作流程

### Step 1：确认行程框架

与用户确认以下信息：
- 起点、终点（是否环线）
- 总天数
- 必去景点 / 途经城市
- 出发季节（影响路况和住宿）

如果用户未指定，主动给出合理的默认建议。

### Step 2：设计每日节点

按天拆分节点，遵循原则：
- 每天驾驶 4-6 小时，不超过 6 小时
- 每天的 points 列表包含当天途经的核心地点
- 终点城市需同时作为下一天的起点（用于里程计算）

### Step 3：调用高德 API

依赖 `personal-map` skill 的 `AMapPersonalMapClient`：

```python
from scripts.amap_personal_map_client import AMapPersonalMapClient
client = AMapPersonalMapClient(api_key=os.getenv("AMAP_API_KEY"))
```

对每个地点执行：
1. `maps_geo(address)` 获取精确坐标
2. `maps_text_search(name, offset=5)` 获取 poiId
3. 验证搜索结果的坐标与 geo 坐标距离 < 30km，防止跨城误匹配
4. 每次 API 调用后 `time.sleep(0.3)`，避免 QPS 超限

### Step 4：驾车路径规划（跨天补全）

计算每日里程时，**必须**把前一天的终点插入当天 points 列表头部作为起点：

```python
for i, day in enumerate(days):
    points = [p for p in day["points"] if "lon" in p]
    if i > 0:
        prev = [p for p in days[i-1]["points"] if "lon" in p]
        if prev:
            points.insert(0, prev[-1])
    # 逐段调用 maps_direction_driving
```

### Step 5：生成个人地图二维码

1. 收集所有有效点（`poiId` 不为空且不为占位符）
2. 去重（按名称），但**保留首尾同名终点**
3. 精简到单条路线 ≤ 16 个节点
4. 调用 `maps_schema_personal_map`，`sceneType=3`（路径规划类）
5. 下载二维码图片并展示

### Step 6：输出 Markdown 行程表

在对话中直接输出 Markdown，包含：
- 行程概览表格（天数、路线、里程、时长、景点）
- 每日详细行程（途经点、坐标、备注）
- 实用提示（高反、门票、路况、温差等）
- 二维码图片和备用链接

## 关键约束

| 约束 | 说明 |
|------|------|
| 单条路线节点上限 | 高德 `lineList` 单条 `pointInfoList` 最多 16 个点 |
| poiId 必填 | `maps_schema_personal_map` 要求 `poiId` 不能为空 |
| poiId 获取策略 | 先用 `maps_text_search` 获取，失败则加城市名再搜，仍失败用占位符 `B000000000` |
| QPS 限制 | 连续调用需间隔 0.3 秒，否则会触发 `CUQPS_HAS_EXCEEDED_THE_LIMIT` |
| API Key | 必须配置环境变量 `AMAP_API_KEY` 或传入构造函数 |

## 输出模板

```markdown
# [路线名称]自驾行程指南

> 起点/终点：[城市]　> 全程预估：约 X km　> 建议天数：X 天

## 行程概览

| 天数 | 路线 | 预估里程 | 预估时长 | 核心景点 |
|------|------|----------|----------|----------|
| Day 1 | ... | ... | ... | ... |

## 每日详细行程

### Day 1：[标题]
- **驾驶里程**：X km
- **驾驶时长**：约 X 小时

**途经点**：
- **名称**：备注 `坐标`

## 实用提示
1. ...

## 个人地图二维码
![路线名称](file:///...)
```

## 进阶能力

- **沿途补给搜索**：用 `maps_around_search` 在关键节点（如无人区前的城镇）搜索加油站、医院、充电桩
- **里程异常排查**：如果某天里程为 0 或明显异常，检查前一天终点是否被正确插入，以及坐标是否重复
- **季节性调整**：7-8 月旺季需提前订房；冬季部分路段（如祁连山）可能封闭，需绕行

## 参考

- 完整示例见 [examples.md](examples.md)
- 底层 API 能力依赖 `personal-map` skill
