# DML 诊断与敏感性分析（审稿硬性要求）

> 本文件是 workflow.md 的补充：把顶刊审稿中 DML 论文"必做"的诊断与敏感性检验落成可执行模板。
> 引擎双轨：Stata（ddml + pystacked + boottest）与 Python 替身（DoubleML + scikit-learn + econml + wildboottest）。

## D1. 遗漏变量敏感性分析（Omnibus partial R²，必做）
- **做什么**：量化"多大的未观测混淆能推翻 θ̂"。DML 不自动解决内生性，必须给边界。
- **方法**：Chernozhukov–Cinelli–Newey–Sharma–Syrgkanis (2024, "Long Story Short") 的 partial R² 框架：
  报告当未观测混淆解释结局残差方差 cf_y、处置残差方差 cf_d、二者相关 rho 时 θ̂ 的偏误与置信区间，
  以及使 θ̂ 归零的临界 cf_y/cf_d。
- **Python**：`dml_obj.sensitivity_analysis(cf_y=0.03, cf_d=0.03, rho=1.0, level=0.95)`（PLR/IRM/PLIV/LPQ 均支持）。
- **Stata**：ddml 无内置，用 Python 结果补；如不可行须在交付物显式说明，不得省略。
- **怎么读**：临界 cf_y 越小（如 1% 的混淆就推翻）→ 结论越脆弱；需 50% 以上混淆才能推翻 → 稳健。

## D2. 重叠性/共同支撑门控（硬性）
- **做什么**：检查 P(D=1|X) 是否远离 0/1，防止估计依赖外推。
- **做法**：①报倾向得分分布（min/p5/p95/max + 直方图）；②修剪极端值（默认 0.05~0.95，可放宽 0.01~0.99）；
  ③报修剪前后 N 与 θ̂ 对照。
- **怎么读**：大量 p̂ 集中在 0 或 1 → 重叠不足，必须修剪并披露；修剪后结论不变才算稳。

## D3. 二元结局 LPQ 的边际效应（APE）
- **做什么**：LPQ 给出对数优势比 θ̂，期刊要求报边际效应。
- **做法**：APE = θ̂ × (1/n)Σ ĝ_i(1−ĝ_i)，ĝ_i 为折外预测的 E[Y|X]（logit）；SE 用 Delta 法或 bootstrap（分位 CI）。
- **注意**：`DoubleMLLPQ` 本版需**二元工具变量 Z**，且 ml_g/ml_m 用**分类器**；无二元 Z 时改用 LPM（`DoubleMLPLR` 对 0/1 Y）。
- **替代**：需稳健结论时补 LPM（`DoubleMLPLR` 对 0/1 Y）作对照，AP 易解释。
- **怎么读**：APE 与对数优势比同号；报告两者并注明口径。

## D4. 异质性推断：BLP 与 GATES（必做，替换"分组各自显著"）
- **做什么**：对估计的 CATE 做有保证的推断，避免"排序分组后回归"的选择偏误。
- **方法**（Chernozhukov–Demirer–Duflo–Fernández-Val 2024）：
  1. BLP：Y 对 CATE 预测值回归（β1=1 表示 CATE 预测准确；β2>0 表示存在异质性）。
  2. GATES：按 CATE 预测值排序分 5~10 组，估计组 ATE，给**同时覆盖所有组的 honest 置信区间**（多重检验校正）。
- **Python**：拟合 IRM 后 BLP 用 `irm.cate(basis=DataFrame)`；GATES 用 `irm.gate(groups=独热DataFrame)`（组列须为 int/bool 哑变量）；底层为 `DoubleMLBLP`。
- **怎么读**：BLP 的 β2 显著 >0 → 确有异质性；GATES 顶组显著高于底组 → 报方向与幅度；勿仅凭"一组显著一组不显著"下结论。

## D5. 聚类稳健推断：wild cluster bootstrap（聚类数少时必做）
- **做什么**：聚类数 < 40~50 时普通聚类 SE 偏小、假显著。
- **Python**：`from wildboottest import wildboottest`（对 DML 得分 ψ̂ 做 WCB）。
- **Stata**：`boottest`（Roodman et al. 2019）或 ddml 的 bootstrap 选项。
- **双向聚类**：`DoubleMLData.from_arrays(x, y, d, cluster_vars=DataFrame[[个体,时间]])` 做双向聚类（`DoubleMLClusterData` 在 0.11 已弃用，改用 `cluster_vars` 参数）。
- **怎么读**：WCB p 与普通聚类 p 一致 → 稳健；不一致以 WCB 为准。

## D6. 连续处置的剂量-反应（ADRF，连续 D 时）
- **做什么**：连续 D 只做线性 PLR 可能掩盖非线性剂量-反应。
- **Python**：`econml.CausalForestDML(discrete_treatment=False)` 或 `econml.dml.NonParamDML` 估计 ADRF 与区间。
- **怎么读**：报剂量-反应曲线 + 置信带；线性段与非线性段分别讨论。

## D7. 计数/截断结局（Y 为计数/截断时）
- **做什么**：Y 为计数（专利/事故）或截断时，需自定义 Neyman 正交 score。
- **说明**：DoubleML 无原生 Poisson/Tobit DML；用 econml 或自定义 score 实现，并在交付物注明"非标准 score，谨慎"。
- **怎么读**：报发生率比/边际效应，注明 score 构造与识别假设。

## D8. 面板 DML 与双向聚类
- **做什么**：面板数据把 FE 与双向聚类纳入 DML。
- **做法**：FE 用组内去均值或虚拟变量；SE 用 `DoubleMLClusterData` 双向聚类；必要时面板 DML（Arkhangelsky 等）。
- **怎么读**：双向与单向聚类的差异反映时间维相关是否重要。

## D9. 分批次（staggered）处置的 DML-DID
- **做什么**：分批次政策用 `DoubleMLDID`/`DoubleMLDIDCS`（含自带敏感性分析），报处置前后各期效应。
- **怎么读**：政策前各期系数不显著且平坦 → 平行趋势；政策后显著 → 因果；`DoubleMLDIDCS` 的敏感性分析给未观测混杂边界。
