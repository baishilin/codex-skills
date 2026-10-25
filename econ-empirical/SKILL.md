---
name: econ-empirical
description: >
  计量经济学实证分析工作流：当用户提供数据与变量信息、模型设定，并要求完成描述统计、
  基准回归、稳健性检验、内生性处理、异质性、机制检验及图表与讨论交付时使用。
  计算引擎优先用 Stata（经 stata skill 的本地 MCP）真跑，Stata 未连接时自动改用
  Python（statsmodels + linearmodels）替身并在结果中明确注明；产出可复现 do 文件或对应 Python 脚本。
  不用于纯数据清洗或纯可视化。
---

# 计量实证分析（econ-empirical）

## 何时使用
- 用户给出数据（.dta/.csv/.xlsx）+ 变量信息 + 模型/研究设定 + 分析要求，要完成整条或部分实证链条。
- 范围：描述统计 → 基准回归 → 稳健性 → 内生性 → 异质性 → 机制 → 讨论与交付。
- 不适用：纯数据清洗/合并（直接用 stata skill 做）、纯可视化。

## 计算引擎（优先 Stata，自动降级 Python）
1. **先探测**：检查当前会话是否有 Stata MCP 工具（工具列表中是否存在 `stata_status` / `stata_run`）。
2. **有 Stata** → 全部走 Stata（见「执行」），这是首选。
3. **无 Stata** → 自动改用 Python 替身引擎：pandas 读数据、statsmodels（OLS/logit/probit 等）、linearmodels（PanelOLS 固定效应/聚类）、scipy.stats.mstats.winsorize（缩尾）、PIL/matplotlib（出图）。**每一步的结果、表格、图、讨论都要明确标注「Python 替身」，绝不冒充 Stata。**
4. 无论哪个引擎，都必须遵守下方「全局规则」；并始终交付一份 Stata 版 do 文件作为正式交付物（用 Python 时另附 Python 脚本）。

## 全局规则（每一轮都生效）
1. **软件环境**：优先用 Stata（经 stata skill 的本地 MCP）；Stata 未连接时自动改用 Python 替身（pandas + statsmodels + linearmodels），并在结果中注明「Python 替身」。用 Stata 时缺包调用 `stata_install_package`（不要自己在 `stata_run` 里塞 `ssc install`）；在 do 文件/脚本与交付说明里记录软件与命令/包版本。
2. **必须真跑（无论哪个引擎）**：每步先写代码（Stata 或 Python）再执行；报错先自行定位修正，最多重试 3 次；最终给出可复现的完整脚本与 log 关键片段。所有报告数字必须来自真实输出，禁止编造、抄错或“合理估计”；某步无法运行或未通过必须如实说明。固定随机种子：Stata 用 `set seed {种子}`，Python 用 `np.random.seed({种子})`。
3. **数据预处理**：正确读入（import/use 或 pandas.read_*）、核对变量类型（字符型 destring/encode 或 astype/categorical）、`xtset {个体} {时间}`（Python 用 MultiIndex 设面板）并报告是否平衡、贴标签、报告关键变量缺失值及处理方式。缩尾只对**连续 Y 与连续 X**，默认双侧 1%：Stata `winsor2 {Y} {X}, replace cuts(1 99)`，Python `scipy.stats.mstats.winsorize`。**不缩尾 D**（0-1/政策变量绝不缩尾；连续强度变量确需缩尾须说明理由）。同时输出：①样本筛选流程图；②变量定义表。
4. **标准误**：默认聚类稳健 `vce(cluster {层级})`（Python 用 linearmodels `cluster_entity` / 聚类协方差）；**聚类层级必须 ≥ D 的变异/处理发生层级**（绝不能低于处理层级，如地区/行业政策不能聚到个体层面）。聚类数 < 40~50 补 wild cluster bootstrap p 值（Stata 用 boottest；Python 替身实现不了时如实说明）。模型不支持聚类时说明理由并改用 vce(robust)（Python 用 HC 稳健协方差）。不同列不同聚类须在表注逐列注明。
5. **显著性与经济显著性**：`*** p<0.01`，`** p<0.05`，`* p<0.1`；括号报 t 值（OLS）或 z 值（logit/probit/IV），保留两位小数；给 95% 置信区间。另报经济显著性：D 一个标准差（基于全样本或处理前口径，避免用缩尾后口径）带来的 Y 变化及其占 Y 均值/标准差的百分比。
6. **表格**：三线表（booktabs 样式）；每模型一列；表头写清被解释变量；表注写明控制变量、固定效应、标准误类型与聚类层级、显著性水平；每列报 N 与 R²（固定效应报组内 R²；logit/probit 报伪 R²）。Stata 优先 `stata_etable`；Python 用 DataFrame 导出 CSV。
7. **每步三段式输出**：①代码 → ②三线表 → ③中文解读 1~3 句（先写“系数符号+显著性”，再写“经济含义”）。
8. **样本量**：每步报 N（面板另报截面数 N_g 与年份数 T）；某步样本量骤减必须说明原因。
9. **多重检验校正**：涉及多个被解释变量、多组异质性或多个机制变量时，**按“检验族”分组**校正（Stata rwolf / Bonferroni / Holm / FDR q；Python 用 statsmodels multipletests），并说明校正前后差异。
10. **识别假设披露**：每步用 1~2 句说明关键识别假设（平行趋势/排他性/共同支撑/条件独立等）及当前数据是否满足；不把相关性结果当因果结论。
11. **诚实披露**：未通过/异常明确说明并给补救建议；披露“做了但未报告”的全部回归，防选择性报告；**用 Python 替身时在交付物与讨论里明确标注，不冒充 Stata。**

## 工作流总览
按顺序执行；不适用步骤可整段删除。每步“做什么 / 方法模型 / 要输出 / 怎么读”的完整模板见 `references/workflow.md`。

0. 数据导入、清洗、面板设定
0.5 **研究设计确认（硬性门控）**
1. 描述性统计
2. 基准回归（2.A 连续 D 默认双向固定效应；2.B 政策/自然实验走 DiD-事件研究）
3. 分维度/分指标回归（可选）
4. 稳健性检验（选 3~5 项）
5. 内生性处理（IV / Oster / PSM / 滞后解释变量 / GMM / RDD 选 1~2）
6. 异质性分析
7. 机制检验（优先直接渠道 D→M；中介谨慎；调节 D×M）
8. 进一步讨论/扩展（可选）

## 第 0.5 步门控（关键）
跑任何回归前，先输出**一页以内的“研究设计摘要”**，包含：识别策略与主要威胁排序、基准模型选择与理由、变量与假设清单、机制类型与检验路径、预期符号，以及**需用户确认的歧义点**（变量度量口径、样本范围、事件/政策时点、聚类层级等）。

输出摘要后**停下等待用户确认或修改**；未确认不得进入回归。

## 输入收集
按 `references/inputs.md` 的清单一次性问全。缺失项用合理默认，并在研究设计摘要中标注“待确认”。

## 执行
- **选引擎**：先探测 Stata MCP 是否可用（有没有 `stata_status`）。可用 → 引擎 A；不可用 → 引擎 B。
- **引擎 A（Stata）**：`stata_status` → 加载数据 → 探索（describe/codebook/summarize）→ 分析 → 导出；图形用 `stata_graph(..., export="outputs/xxx.png", replace=True)`；数据路径正斜杠、变量名大小写敏感、多命令链首错即停，修正第一条错误再重试。
- **引擎 B（Python 替身）**：pandas 读入（.dta→pd.read_stata、.csv→pd.read_csv、.xlsx→pd.read_excel）；面板固定效应/聚类用 linearmodels 的 `PanelOLS(..., entity_effects=True, time_effects=True).fit(cov_type="clustered", cluster_entity=True)`；描述统计与相关用 pandas/statsmodels；缩尾用 scipy.stats.mstats.winsorize；出图用 PIL 或 matplotlib。所有输出与讨论标注「Python 替身」。
- 图表与结果写入用户指定输出目录（默认 `./outputs`）。

## 交付物（最终必须给出）
1. 完整可运行 do 文件（Stata 版，正式交付物；若用 Python 替身则另附 run_analysis.py）——含注释、随机种子、数据路径、全部回归与图形导出，并说明运行环境。
2. 实证结果汇总表：步骤 | 被解释变量 | D 系数（或交互项）| 标准误 | 显著性 | N | 是否支持假设。
3. 假设判定矩阵：H1/H2… | 对应表/图 | 方向与显著性 | 是否支持 | 与文献一致性。
4. 图表清单与文件（事件研究图、coefplot、断点图/简单斜率图、样本筛选流程图、变量定义表等）。
5. 中文“实证结果与讨论”3~5 段：结论 → 基准 → 稳健性 → 内生性 → 异质性 → 机制 → 与文献/假设一致性 → 不足与局限。
6. 未通过/异常结果的如实说明与补救建议。
