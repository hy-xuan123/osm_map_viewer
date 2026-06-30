# OSM-Vis

## Abstract

An open-source, web-based platform for OSM data acquisition, storage, road junction extraction, and interactive visualization. Implements a complete pipeline from data download to web rendering.

---

## 1. Pipeline Overview

**Data Acquisition → Data Storage → Junction Extraction → Web Visualization**

| Stage | Scripts | Output |
|-------|---------|--------|
| Acquisition | `download_guigang_park.py`, `interactive_download.py` | `osm_data/{lat}_{lon}_{alias}/` |
| Storage | (built into download script) | `roads.json`, `combined.json`, `metadata.json`, `index.json` |
| Extraction | `extract_junctions.py`, `gen_junction_js.py` | `guigang_junctions.json`, `junction_data.js`, `intersection_data.js` |
| Visualization | `index.html` | Leaflet.js interactive map |

---

## 2. Data Acquisition

### 2.1 `download_guigang_park.py`

Downloads OSM data via Overpass API. Accepts command-line arguments:

```bash
python download_guigang_park.py <lat> <lon> <expand> <alias>
```

**Download procedure:**

1. Parse arguments (lat, lon, expand, alias)
2. Compute bounding box: `north/south = lat ± expand`, `east/west = lon ± expand`
3. Query 6 categories via Overpass QL: `roads`, `buildings`, `water`, `landuse`, `pois`, `traffic_signals`
4. Try 3 API endpoints sequentially (fault tolerance)
5. Save each category as separate `.json` file
6. Save `combined.json` (all categories merged)
7. Save `metadata.json` (coordinates, range, statistics)
8. Update `osm_data/index.json` (dataset index)

### 2.2 `interactive_download.py`

Wraps `download_guigang_park.py` as a subprocess. Provides interactive CLI with coordinate input prompts and download confirmation.

---

## 3. Data Storage

### 3.1 Directory Structure

```
osm_data/
└── {lat}_{lon}_{alias}/
    ├── roads.json
    ├── buildings.json
    ├── water.json
    ├── landuse.json
    ├── pois.json
    ├── traffic_signals.json
    ├── combined.json
    └── metadata.json
```

### 3.2 `metadata.json` Schema

```json
{
  "download_time": "ISO 8601",
  "center_coordinates": {"latitude": float, "longitude": float},
  "download_range": {"north": float, "south": float, "east": float, "west": float, "expand": float},
  "alias": "string | null",
  "data_categories": ["roads", "buildings", ...],
  "statistics": {"roads": {"ways": int, "nodes": int, "total_elements": int}, ...}
}
```

### 3.3 `osm_data/index.json` Schema

```json
{
  "datasets": [
    {"lat": float, "lon": float, "alias": "string", "directory": "string", "path": "string", "download_time": "ISO 8601"}
  ]
}
```

---

## 4. Junction Extraction

### 4.1 `extract_junctions.py`

Extracts road junctions from `roads.json`.

**Logic:**

1. Separate `ways` and `node_coords` from OSM elements
2. Count way references per node (`node_way_count`: `dict[node_id] → set[way_id]`)
3. Filter nodes referenced by ≥2 ways → **junctions**
4. Among junctions, filter those with ≥2 `MAIN_HIGHWAYS` (primary/secondary/tertiary) → **intersections**
5. Sort by `way_count` descending
6. Save to `guigang_junctions.json`

### 4.2 `gen_junction_js.py`

Reads `guigang_junctions.json`. Generates `junction_data.js` and `intersection_data.js` as JavaScript source files for direct frontend loading.

### 4.3 `estimate_junctions.py`

Estimates deduplicated junction count using Haversine distance threshold (default 50m). Reads a `roads.json` file, computes raw/deduplicated counts, and prints time estimates for processing.

---

## 5. Web Visualization (`index.html`)

### 5.1 Data Loading Strategy

Priority order:

1. `currentDataset` (global variable, set after user selects dataset)
2. `currentOsmPath` (global variable)
3. Fetch `osm_data/index.json`, load first dataset

For each dataset: fetch `combined.json` → parse OSM elements → render layers.

### 5.2 Rendering Layers

| Layer | Data | Visual |
|-------|------|--------|
| Roads | `roads.json` | Color by highway class |
| Buildings | `buildings.json` | Pink semi-transparent polygons |
| Water | `water.json` | Blue semi-transparent polygons |
| Landuse | `landuse.json` | Color by landuse type |
| Junctions | `junction_data.js` | Red circle markers |
| Intersections | `intersection_data.js` | Orange diamond markers |
| Traffic Signals | `traffic_signals.json` | Green icon markers |

### 5.3 Interactive Features

| Feature | Trigger | Description |
|---------|---------|-------------|
| Search | `Ctrl+F` / search button | Search roads, junctions, or coordinates |
| Fly to Junction | Click search result | Fly animation, pan offset for info panel, persistent red marker with pulse |
| Coordinate Inspection | "Coordinates" button | Hover on junction shows floating panel with coords and road names |
| Map Pick | "Map Pick" button in search dialog | Click map to capture coordinates into search input |
| Toggle Layers | Layer buttons (bottom-left) | Toggle roads/buildings/water/landuse/junctions/intersections/traffic signals |
| Save Markers | Save button in search dialog | Save user-annotated markers to `saved_markers/` |

### 5.4 Traffic Signal Loading

When "Traffic Signals" layer is toggled on:

1. Try `traffic_signals.json` in current dataset directory
2. If not found, extract from `combined.json`
3. Render markers on map

---

## 6. Usage

**Download new data:**
```bash
python download_guigang_park.py 23.1122 109.5990 0.009 guigang_park
```

**Extract junctions:**
```bash
python extract_junctions.py
python gen_junction_js.py
```

**Run web interface:**
```bash
python -m http.server 8080
# Open http://localhost:8080
```

---

## 7. Technical Specifications

| Item | Detail |
|------|--------|
| Frontend | Leaflet.js 1.9.4 |
| Backend | Python 3 |
| Data Source | OpenStreetMap (Overpass API) |
| Coordinate System | WGS84 (EPSG:4326) |
| Distance Calculation | Haversine formula, R = 6371000 m |

---

## License

MIT

---

# OSM-Vis（中文版）

## 摘要

一个开源、基于 Web 的 OSM 数据获取、存储、道路交叉口提取与交互式可视化平台。实现了从数据下载到 Web 渲染的完整流水线。

---

## 1. 流水线概览

**数据获取 → 数据存储 → 交叉口提取 → Web 可视化**

| 阶段 | 脚本 | 输出 |
|------|------|------|
| 获取 | `download_guigang_park.py`、`interactive_download.py` | `osm_data/{lat}_{lon}_{别名}/` |
| 存储 | （内置于下载脚本） | `roads.json`、`combined.json`、`metadata.json`、`index.json` |
| 提取 | `extract_junctions.py`、`gen_junction_js.py` | `guigang_junctions.json`、`junction_data.js`、`intersection_data.js` |
| 可视化 | `index.html` | Leaflet.js 交互式地图 |

---

## 2. 数据获取

### 2.1 `download_guigang_park.py`

通过 Overpass API 下载 OSM 数据。接受命令行参数：

```bash
python download_guigang_park.py <纬度> <经度> <范围> <别名>
```

**下载流程：**

1. 解析参数（纬度、经度、范围、别名）
2. 计算边界框：`north/south = 纬度 ± 范围`，`east/west = 经度 ± 范围`
3. 通过 Overpass QL 查询 6 个分类：`roads`、`buildings`、`water`、`landuse`、`pois`、`traffic_signals`
4. 依次尝试 3 个 API 端点（容错机制）
5. 将每个分类保存为独立的 `.json` 文件
6. 保存 `combined.json`（所有分类合并）
7. 保存 `metadata.json`（坐标、范围、统计信息）
8. 更新 `osm_data/index.json`（数据集索引）

### 2.2 `interactive_download.py`

将 `download_guigang_park.py` 封装为子进程调用，提供交互式命令行界面，包含坐标输入提示和下载确认。

---

## 3. 数据存储

### 3.1 目录结构

```
osm_data/
└── {纬度}_{经度}_{别名}/
    ├── roads.json
    ├── buildings.json
    ├── water.json
    ├── landuse.json
    ├── pois.json
    ├── traffic_signals.json
    ├── combined.json
    └── metadata.json
```

### 3.2 `metadata.json` 结构

```json
{
  "download_time": "ISO 8601 格式",
  "center_coordinates": {"latitude": 浮点数, "longitude": 浮点数},
  "download_range": {"north": 浮点数, "south": 浮点数, "east": 浮点数, "west": 浮点数, "expand": 浮点数},
  "alias": "字符串 | null",
  "data_categories": ["roads", "buildings", ...],
  "statistics": {"roads": {"ways": 整数, "nodes": 整数, "total_elements": 整数}, ...}
}
```

### 3.3 `osm_data/index.json` 结构

```json
{
  "datasets": [
    {"lat": 浮点数, "lon": 浮点数, "alias": "字符串", "directory": "字符串", "path": "字符串", "download_time": "ISO 8601"}
  ]
}
```

---

## 4. 交叉口提取

### 4.1 `extract_junctions.py`

从 `roads.json` 中提取道路交叉口。

**逻辑：**

1. 从 OSM 元素中分离 `ways` 和 `node_coords`
2. 统计每个节点被道路引用的次数（`node_way_count`: `dict[node_id] → set[way_id]`）
3. 筛选出被 ≥2 条道路共享的节点 → **junctions**（交叉口）
4. 在交叉口中，筛选出包含 ≥2 条主干道（primary/secondary/tertiary）的节点 → **intersections**
5. 按 `way_count` 降序排序
6. 保存至 `guigang_junctions.json`

### 4.2 `gen_junction_js.py`

读取 `guigang_junctions.json`，生成 `junction_data.js` 和 `intersection_data.js` 作为 JavaScript 源文件，供前端直接加载。

### 4.3 `estimate_junctions.py`

使用 Haversine 距离阈值（默认 50 米）估算去重后的交叉口数量。读取指定的 `roads.json` 文件，计算原始/去重数量，并输出处理时间估算。

---

## 5. Web 可视化（`index.html`）

### 5.1 数据加载策略

优先级顺序：

1. `currentDataset`（全局变量，用户选择数据集后设置）
2. `currentOsmPath`（全局变量）
3. 获取 `osm_data/index.json`，加载第一个数据集

对每个数据集：获取 `combined.json` → 解析 OSM 元素 → 渲染图层。

### 5.2 渲染图层

| 图层 | 数据来源 | 视觉效果 |
|------|----------|----------|
| 道路 | `roads.json` | 按道路等级着色 |
| 建筑 | `buildings.json` | 粉色半透明多边形 |
| 水域 | `water.json` | 蓝色半透明多边形 |
| 土地利用 | `landuse.json` | 按土地利用类型着色 |
| 交叉口 | `junction_data.js` | 红色圆形标记 |
| Intersection | `intersection_data.js` | 橙色菱形标记 |
| 交通信号灯 | `traffic_signals.json` | 绿色图标标记 |

### 5.3 交互功能

| 功能 | 触发方式 | 说明 |
|------|----------|------|
| 搜索 | `Ctrl+F` / 搜索按钮 | 搜索道路、交叉口或坐标 |
| 飞行定位 | 点击搜索结果 | 飞行动画，为信息面板偏移，持久红色标记带脉动效果 |
| 坐标查看 | "Coordinates"按钮 | 悬停在交叉口上显示浮动面板，展示坐标和道路名称 |
| 地图取点 | 搜索对话框中"Map Pick"按钮 | 点击地图将坐标捕获到搜索输入框 |
| 图层切换 | 图层按钮（左下角） | 切换道路/建筑/水域/土地利用/交叉口/Intersection/交通信号灯图层 |
| 保存标记 | 搜索对话框中保存按钮 | 将用户标注的标记保存至 `saved_markers/` |

### 5.4 交通信号灯加载

当切换"Traffic Signals"图层为开启状态时：

1. 尝试加载当前数据集目录中的 `traffic_signals.json`
2. 若未找到，从 `combined.json` 中提取
3. 在地图上渲染标记

---

## 6. 使用方法

**下载新数据：**

```bash
python download_guigang_park.py 23.1122 109.5990 0.009 guigang_park
```

**提取交叉口：**

```bash
python extract_junctions.py
python gen_junction_js.py
```

**运行 Web 界面：**

```bash
python -m http.server 8080
# 打开 http://localhost:8080
```

---

## 7. 技术规格

| 项目 | 详情 |
|------|------|
| 前端 | Leaflet.js 1.9.4 |
| 后端 | Python 3 |
| 数据来源 | OpenStreetMap（Overpass API） |
| 坐标系统 | WGS84（EPSG:4326） |
| 距离计算 | Haversine 公式，R = 6371000 米 |

---

## 开源协议

MIT
