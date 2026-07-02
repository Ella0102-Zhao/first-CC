# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

单人旅行规划器 — 单文件 HTML 应用（`index.html`），无构建步骤、无依赖、无后端。所有 CSS 和 JS 内联在单个文件中。直接在浏览器中打开即可运行。

## How to run

```bash
open "index.html"
```

无需 `npm install`、构建或服务器。所有数据通过 `localStorage` 持久化。

## Architecture

### 单文件 SPA 模式

整个应用是手写的 vanilla JS 单页应用（无框架），包含三个部分：
- `<style>` — 所有 CSS（CSS 自定义属性、响应式布局、动画）
- `<body>` — 侧边栏 + 模态容器 + Toast 容器的静态 HTML
- `<script>` — 状态管理、CRUD 操作、渲染引擎、API 调用

### 状态管理

`state` 对象通过 `localStorage`（key: `travel-planner-pro-v2`）持久化：

```
state = {
  trips: [Trip],           // 所有旅行计划
  activeTripId: string,    // 当前选中的旅行 ID
  currentView: string,     // 'dashboard' | 'explore' | 'itinerary' | 'budget' | 'packing'
  llmConfig: { provider, apiKey, model }  // LLM API 配置
}
```

`Trip` 对象结构：
```
{
  id, name, country, startDate, endDate,
  budget, budgetType,           // budgetType: 'personal' | 'team'
  travelers, notes, tripType,   // tripType: 'city' | 'beach' | 'mountain' | 'business'
  travelStyles: [],             // 可选值见 TRAVEL_STYLES 常量
  itinerary: { "Day 1": [Activity], ... },
  expenses: [Expense],
  packing: { category: [{id, name, checked}], ... }
}
```

### 视图系统

`render()` 根据 `state.currentView` 分发到不同的渲染函数：
- `renderDashboard()` — 旅行卡片网格
- `renderExplore()` — 目的地信息、天气、照片、AI 生成按钮
- `renderItinerary()` — 按天标签页，拖拽排序活动
- `renderBudget()` — 4 列预算概览 + 分类支出
- `renderPacking()` — 分类行李清单

导航通过 `navigate(view)` 触发，更新 `state.currentView` 后调用 `render()`。

### 外部 API 集成

| 功能 | 数据源 | 说明 |
|------|--------|------|
| 目的地介绍 | Wikipedia REST API | 中文优先，fallback 英文 |
| 目的地照片 | Wikipedia + Unsplash | 缓存到 `photoCache` |
| 天气预报 | weather.com.cn → Open-Meteo | 两层 fallback |
| AI 行程生成 | DeepSeek API（默认，免费） | 也支持 Anthropic、OpenAI |

所有 API 调用都有内存缓存（`weatherCache`、`wikiCache`、`photoCache`、`guideCache`），通过 `refreshExplore()` 清除。

### 关键函数

- `createTrip(data)` / `updateTrip(id, data)` — 行程 CRUD（包含日期变化时的行程天数调整）
- `addActivity(tripId, day, activity)` / `updateActivity(...)` / `deleteActivity(...)` — 活动 CRUD
- `generateAIPlan(trip, onStatus)` — 构建 prompt 并调用 LLM API，解析 JSON 响应填充行程
- `generateAIPlanForTrip(tripId)` — 包装函数，显示 loading modal 后调用 `generateAIPlan`
- `fetchWeather(locationName, startDate, endDate)` — 先尝试 weather.com.cn，失败则 Open-Meteo，都失败返回 `{error, message: '神机妙算失效了...'}`
- `fetchWeatherCN(locationName)` — 通过内置城市代码表查询 weather.com.cn
- `showTripForm(trip?)` — 创建/编辑旅行模态框（含预算类型和旅行风格多选）
- `showSettingsModal()` — LLM API 设置模态框

### 模态框模式

所有模态框通过 `showModal(title, bodyHtml, footerHtml)` 渲染到 `#modal-container`。按钮事件在 `setTimeout` 中绑定以确保 DOM 就绪。`closeModal()` 清除容器内容。
