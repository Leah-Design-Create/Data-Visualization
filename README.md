# ACECQA 教育与照护服务分析（初步探索性数据分析）

本项目使用 **Education and Care services approvals** 的公开数据（来自 NQA ITS 每日更新）对澳大利亚获批教育与照护服务机构进行初步探索性分析，目的是为 **ACECQA（Australian Children’s Education & Care Quality Authority）** 的决策提供数据支持。

分析聚焦三条业务相关的视角：
1. **服务质量（Service Quality）**：整体质量评级在各州的分布情况，以及低质量（例如 *Working Towards NQS*）集中在哪些质量领域（Quality Areas）。
2. **可及性与覆盖（Accessibility and Coverage）**：服务在城市/乡村区域的分布差异，为资源与支持的优先级提供线索。
3. **运营趋势（Operational Trends）**：按批准年份（Approval Year）汇总的容量（Approved places）变化，用于判断容量扩张/不足的时间演化。

> 本仓库目前仅包含 notebook；图表输出文件由运行 notebook 生成（见下方 Output Artifacts）。

---

## 仓库文件

- `540339523_QBUS6860_Individual_Assignment.ipynb`：完成数据读取、清洗、分析与可视化的主脚本（Jupyter Notebook）。
- `Visualization.ipynb`：完成数据读取、清洗、分析与可视化的主脚本（Jupyter Notebook）。

---

## 数据来源与输入文件

Notebook 里实际读取的主要文件如下（请确保放在仓库根目录，或按你自己的路径修改 notebook 中的文件名/读取路径）：

由于隐私与合规原因，原始数据文件不会在 GitHub 仓库中对外公布；你需要自行下载数据集并放置到对应路径后再运行 notebook 生成图表。

### 1) 主数据（服务与审批）

- `Education-services-au-export.csv`

使用到的关键字段（来自 notebook 的处理逻辑）：
- `ServiceApprovalGrantedDate`：用于提取 `ApprovalYear`
- `OverallRating`：整体质量评级
- `QualityArea1Rating` ~ `QualityArea7Rating`：7 个质量领域的评级
- `State`：州/地区缩写（如 `NSW`, `VIC` 等）
- `Suburb`：用于与 SUA 数据合并（匹配到 `SA2_NAME_2021`）
- `NumberOfApprovedPlaces`：用于容量（Total approved capacity）汇总

### 2) 城市/乡村划分（SUA / SA2）

- `SUA_2021_AUST.CSV`

使用到的关键字段（来自 notebook 的处理逻辑）：
- `SA2_NAME_2021`：与 `Suburb`（均转为大写）匹配
- `SUA_NAME_2021`：用于判断 `Region_Type`（urban/rural）
- `Region_Type`：由程序生成（`Not in any Significant Urban Area` -> `rural`，否则 `urban`）

Notebook 同时会过滤掉与 `SUA_NAME_2021` 包含 `Outside Australia` 相关的记录。

### 3) 州/地区空间边界（Shapefile）

- `SA4_2021_AUST_GDA2020.shp`（以及配套文件：通常还需要 `.shx`、`.dbf`、`.prj`）

使用到的关键字段（来自 notebook 的处理逻辑）：
- `STE_NAME21`：先 dissolve 成州级别边界后用于 choropleth 地图

---

## 方法概述（Notebook 中的核心处理步骤）

### A. 服务质量分析（Service Quality）

1. **整体质量评级数值化**
   - 将 `OverallRating` 的类别映射为数值（用于均值/可视化对比），映射规则在 notebook 中定义为：
     - `Significant Improvement Required` -> 1
     - `Working Towards NQS` -> 2
     - `Meeting NQS` -> 3
     - `Exceeding NQS` -> 4
     - `Excellent` -> 5

2. **按州展示 OverallRating 分布**
   - 对每个州提取 `OverallRating_num` 后用 violin plot 绘制，并保存为：
     - `OrverallRating across States.png`

3. **低分质量领域热力图（Working Towards NQS）**
   - 将 `QualityArea1Rating` ~ `QualityArea7Rating` 转成长表（melt），筛选出评级为 `Working Towards NQS` 的记录。
   - 再按 `(State, QualityArea)` 统计低分出现次数，pivot 成矩阵并绘制热力图，保存为：
     - `LowRating per Quality Area by State (Working Toward NQS).png`

4. **质量领域评分随审批年份变化（2008-2025）**
   - 将 `ServiceApprovalGrantedDate` 解析为日期并提取 `ApprovalYear`
   - 筛选 `ApprovalYear` 在 `2008 ~ 2025`
   - 将每个 Quality Area 的评级同样数值化（`Q{i}_num`），对每一年与每个 Quality Area 计算平均分
   - 使用 lineplot 可视化趋势，并保存为：
     - `Trend of Quality Area Ratings by Approval Year.png`

### B. 可及性与覆盖（Accessibility and Coverage）

1. **城市/乡村（urban/rural）定义与合并**
   - 从 `SUA_2021_AUST.CSV` 中生成 `Region_Type`：
     - `SUA_NAME_2021` 包含 `Not in any Significant Urban Area` -> `rural`
     - 否则 -> `urban`
   - 将 `SUA` 的 `SA2_NAME_2021` 转为大写，并与服务表的 `Suburb`（同样需保持匹配口径）合并：
     - `pd.merge(df, sua, left_on='Suburb', right_on='SA2_NAME_2021', how='left')`

2. **城市/乡村分布对比**
   - 使用 countplot 按州（`State`）统计 `Region_Type` 的服务数量，并保存：
     - `Urban vs Rural Service Distribution by State.png`

3. **州级服务数量空间可视化（交互式地图）**
   - 统计每个 `State` 的服务数量（ServiceCount）
   - 使用 shapefile dissolve 成州级边界，并用 Plotly `choropleth_mapbox` 生成交互式地图
   - 保存为：
     - `Number of Education Services by State.html`

### C. 运营趋势（Operational Trends）

1. **从审批日期提取 Approval Year**
   - `ServiceApprovalGrantedDate` -> datetime -> `.dt.year` -> `ApprovalYear`

2. **按州汇总批准容量（Approved places）并随时间变化**
   - 按 `["ApprovalYear", "State"]` 汇总 `NumberOfApprovedPlaces` 的和，生成：
     - `df_capacity`，包含字段 `TotalCapacity`
   - 为保证动画维度完整，构造所有 `(year, state)` 组合的 MultiIndex，并对缺失组合填充 0
   - 使用 Plotly `px.bar` + `animation_frame="ApprovalYear"` 制作动画图，并保存为：
     - `Total Approved Capacity by State OVer Time.html`

---

## 输出产物（Output Artifacts）

运行 notebook 后，项目根目录会生成以下文件：

### 静态图片（PNG）
- `OrverallRating across States.png`
- `LowRating per Quality Area by State (Working Toward NQS).png`
- `Trend of Quality Area Ratings by Approval Year.png`
- `Urban vs Rural Service Distribution by State.png`

### 交互式图（HTML）
- `Number of Education Services by State.html`
- `Total Approved Capacity by State OVer Time.html`

---

## 如何复现实验（Reproducibility）

### 1) 安装依赖

建议使用 Python 环境（至少需要以下库）：
- `pandas`
- `numpy`
- `matplotlib`
- `seaborn`
- `plotly`
- `geopandas`

如果你运行失败与 `geopandas` 的底层依赖相关，请按你系统环境补齐（常见需要 `fiona` / `shapely` / `pyproj` 等）。

### 2) 准备输入数据

把 notebook 里引用的文件放到仓库根目录：
- `Education-services-au-export.csv`
- `SUA_2021_AUST.CSV`
- `SA4_2021_AUST_GDA2020.shp` 以及配套 shapefile 文件

### 3) 运行方式

打开并运行：
- `540339523_QBUS6860_Individual_Assignment.ipynb`

---

## 业务问题动机（为什么这些图对 ACECQA 有用）

- **服务质量**：通过整体评级分布与低分质量领域热力图，可以快速定位在不同州哪些质量领域更容易出现 *Working Towards NQS*，从而帮助 ACECQA 更有针对性地设计支持与监管优先级。
- **可及性与覆盖**：城市/乡村差异与州级空间分布能帮助理解服务供给的结构性不均衡，利于资源配置与进一步的现场核查/教育支持。
- **运营趋势**：容量随批准年份的演化可为规划提供“随时间增长/停滞”的证据，帮助判断哪些州在历史上扩张较快、哪些州可能存在供给不足风险。

---

## 后续可改进点（可选）

- 将服务质量进一步细分到 `ServiceType`（如果你能获取/确认该字段对应的质量维度），以满足作业“按服务类型”的扩展要求。
- 增加质量与容量之间的相关性/分组对比（例如按州将低分质量领域强度与容量增长速率关联）。

