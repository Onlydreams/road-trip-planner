# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个 Claude Code skill 仓库，用于基于高德地图 API 规划多日自驾游路线（仅限中国大陆）。Skill 入口为 `SKILL.md`，当用户提及自驾、公路旅行、路线规划时自动触发。

依赖 [@lbs-amap/personal-map](https://clawhub.ai/lbs-amap/personal-map) skill，需配置环境变量 `AMAP_API_KEY`。

## 架构

- `SKILL.md` — skill 定义（触发条件、6 步工作流、输出模板、约束表）
- `examples.md` — 三个完整案例（蒙东 8 日、青甘 8 日、川西 5 日），包含里程表、节点列表和跨天补全说明
- `README.md` / `README_CN.md` — 中英文项目说明

## 工作流（6 步）

1. 与用户确认行程框架（起终点、天数、必去景点、季节）
2. 按天拆分节点，每日驾驶 ≤ 6 小时，高原/山区进一步压缩
3. 调用高德 API（依赖 `personal-map` skill 的 `AMapPersonalMapClient`）：`maps_geo` → `maps_text_search` → 坐标验证（距离 < 30km）→ sleep 0.3s
4. **跨天里程补全**：计算每日里程时必须把前一天终点插入当天 points 头部作为起点
5. 生成个人地图二维码：去重（按名称，保留首尾同名终点）→ 精简至 ≤ 16 节点 → `maps_schema_personal_map(sceneType=3)`
6. 输出 Markdown 行程表（概览表格 + 每日详情 + 实用提示 + 二维码）

## 关键约束

- 单条路线最多 16 个节点（高德限制）
- poiId 不能为空，获取失败时用占位符 `B000000000`
- API 调用间隔 ≥ 0.3 秒
- 环境变量 `AMAP_API_KEY` 必须配置

## 常见问题

- **某天里程为 0**：检查是否遗漏跨天补全（前一天终点未插入当天头部）
- **坐标跨城误匹配**：`maps_text_search` 结果需与 `maps_geo` 坐标做距离验证
- **环线首尾去重**：去重时保留首尾同名终点，否则路线不闭合
