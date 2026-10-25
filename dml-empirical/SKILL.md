---
name: dml-empirical
description: >
  双重机器学习（Double/Debiased Machine Learning, DML）因果推断实证分析工作流。
  当用户提供数据与变量信息（被解释变量 Y、处置变量 D，可选控制变量/机制/工具变量），
  要求以双重机器学习完成描述统计、基准回归、稳健性、内生性（部分线性工具变量）、
  异质性、机制（因果中介）检验与图表讨论交付时使用；只需数据与变量信息即可自动完成整条实证链。
  引擎优先 Stata（ddml 包经 stata MCP），Stata 未连接时自动改用 Python（DoubleML + scikit-learn）并标注「Python 替身」。
  不用于纯数据清洗或纯可视化。
metadata:
  short-description: 双重机器学习实证分析
---

# 双重机器学习因果推断实证分析（dml-empirical）

## 何时使用
- 用户给出数据 + 变量信息（Y、D，可选 X/M/Z）+ 研究设定，要求做 DML 因果推断实证（整条或部分）。
- 政策/干预效果评估、高维/非线性控制、明确要求 DML/正交化/交叉拟合、或需在同一框架完成
  「基准 → 稳健性 → 内生性 → 异质性 → 机制」。
- 不适用：纯数据清洗/合并、纯可视化、以传统 DID 平行趋势/事件研究为主的分析（用 econ-empirical）。

## 自动完成承诺（本 skill 的核心）
- 用户只需提供：**数据路径 + 被解释变量 Y + 处置变量 D + 样本结构**；其余（控制变量、固定效应、
  聚类层级、学习器、交叉拟合折数、缩尾、滞后、输出目录）**自动推断或取默认**，并写进最终交付物的「假设与默认清单」。
- 仅在“无法安全推断且会实质改变结论”的歧义点停下来询问；否则一路跑完。

## 计算引擎
1. **先探测**：是否存在 Stata MCP 工具（`stata_status` / `stata_run`）。
2. **有 Stata** → `ddml` 包（缺包用 `stata_install_package` 装 `ddml`、`pystacked`、`lassopack`；`pystacked` 需本机 Python + scikit-learn，否则用 `lassocv`）。
3. **无 Stata** → Python 替身 `doubleml`（DoubleMLPLR/IRM/PLIV/LPQ/DID）+ `scikit-learn`，**所有输出标注「Python 替身」**。
4. 两引擎都必须交付可复现 do 文件（Python 替身另附 .py）。

## 最小输入与自动推断
详见 `references/inputs.md`。核心：只给「数据路径 + Y + D + 样本结构」即可启动；
D/Y 类型、X、FE、聚类、学习器、K 折、缩尾、滞后均按表内规则自动判定或取默认。

## 核心约束（速记；完整版见 references/workflow.md「全局规则」）
1. 必须真跑：先写代码再执行，报错最多修 3 次，数字全部来自真实输出，禁止编造。
2. 固定随机种子与交叉拟合 folds，保证可复现。
3. 缩尾只对连续 Y/X（双侧 1%），**D 绝不缩尾**；滞后一期须注明。
4. 二元 D 的辅助回归用分类器，连续 D 用回归器；Y 二元用 LPQ/logit 或 LPM。
5. 交叉拟合折内调参、折外预测，**严禁全样本训练后全样本预测**（信息泄漏 → 标准误虚小）。
6. 聚类层级 ≥ D 的变异/处理层级；标准误基于正交化得分。
7. 稳健性须同时覆盖数据层与模型层（≥3~5 项），政策类补安慰剂检验。
8. 每步三段式输出：代码 → 三线表 → 中文解读；披露全部识别假设，不把相关当因果。**三线表必须严格按论文期刊格式（顶线粗/栏目线细/底线粗、无竖线），且必须是可复制/可编辑的文本表格（Markdown/Word 原生/LaTeX/CSV），禁止以 PNG 图片作为结果表——见 `references/tables.md`。**
9. 用 Python 替身时在交付物与讨论中明确标注，不冒充 Stata。
10. 遗漏变量敏感性分析（Omnibus partial R²，Chernozhukov–Cinelli 2024）为**必做**项，量化未观测混淆边界（`sensitivity_analysis()`）。
11. 重叠性/共同支撑是**硬性门控**：报倾向得分分布、极端值修剪、修剪前后 N 与 θ̂ 对照。
12. 异质性推断用 **BLP/GATES**（`DoubleMLBLP`）+ 排序组 honest CI + 多重检验，勿仅凭分组各自显著下结论。
13. 聚类数 < 40~50 必须 **wild cluster bootstrap**（`wildboottest`/`boottest`），面板双向聚类用 `DoubleMLData.from_arrays(..., cluster_vars=[个体,时间])`（`DoubleMLClusterData` 已弃用）。
14. 每步显式披露识别假设与 DML 渐近条件：无混淆（D⊥U|X）、重叠（0<P(D=1|X)<1）、Neyman 正交性、nuisance 收敛率 n^{-1/4}；PLIV 另加相关性+排他性。

## 工作流总览（详见 references/workflow.md；诊断与敏感性见 references/diagnostics.md）
0. 数据导入/清洗/面板设定 → 0.25 复合指标测算（可选）→ 0.5 研究设计摘要（门控）→ 1 描述统计
→ 2 基准回归（PLR，按 D×Y 类型；重叠性门控）→ 3 分维度 → 4 稳健性（数据层+模型层+安慰剂+遗漏变量敏感性）
→ 5 内生性（PLIV/弱IV/LATE）→ 6 异质性（分组/交互/CATE+BLP/GATES）→ 7 机制（因果中介）→ 8 扩展（DML-DID/剂量-反应）。

## 第 0.5 步门控
- 输入完整、歧义可自动化解 → **自动通过**，在交付物中列明所有默认与假设，不再停等。
- 仅当存在会实质影响结论的歧义（变量口径、政策时点、聚类层级、是否滞后等）时，停下询问一次。

## 交付物（最终必须给出）
1. 可运行 do 文件（Stata 版；Python 替身另附 .py）：注释、种子、数据路径、全部回归与图形导出。
2. 实证结果汇总表：步骤 | 被解释变量 | θ̂ | 标准误 | 显著性 | 95% CI | N | 是否支持假设。
3. 假设判定矩阵：H1/H2… | 对应表/图 | 方向与显著性 | 是否支持 | 与文献一致性。
4. 图表清单（描述统计表、coefplot、重叠性诊断图、机制分解图、异质性图、安慰剂图、样本筛选流程图、变量定义表）；其中所有三线表严格遵循期刊格式（`references/tables.md`），**以可复制文本（Markdown/Word 原生三线表/LaTeX/CSV）交付，结果表禁止用 PNG 图片**；图片仅用于真正的图。
5. 中文“实证结果与讨论”3~5 段：结论→基准→稳健性→内生性→异质性→机制→与文献一致性→不足。
6. 「假设与默认清单」+ 未通过/异常结果的如实说明与补救建议。

## 参考文件
- `references/workflow.md`：全局规则（完整版）+ 步骤 0~8 模板 + 附录（模型变体/诊断/论文映射/审稿人清单/自检）。
- `references/inputs.md`：最小输入、自动推断与默认值、一次性提问模板。
- `references/tables.md`：三线表期刊级格式规范（硬性）。
- `references/diagnostics.md`：DML 诊断与敏感性分析（遗漏变量 partial R²、重叠性门控、BLP/GATES、wild bootstrap、剂量-反应、计数/截断、面板 DML、DML-DID）。
