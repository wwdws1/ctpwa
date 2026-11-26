# CHANGELOG（`example/`）

本文件记录对上游 `BHXiang/ctpwa` 的改动中、**`example/` 目录内**各文件的内容变化
（PR #1–#26）。其它目录通常随上游维护，不在本表；
唯一例外是 `setup.py` 的 C++20 改动（见 2026-11-24，本 fork 与上游的第一处非 `example/` 分歧）。
**组织方式：先按合并进 `main` 的日期（`##`）分组，日期内再按文件（`###`）分组**；
不涉及 `example/` 的上游同步提交也按日期归入各日期的「上游同步」小节。
“PR #n”指 `wwdws1/ctpwa` 的合并请求。

> 只列**行为/接口/文档**变化；纯格式（ruff-format/空白）只在首次记录。

---

## 2026-11-25

### 上游同步（v0.3.25 + v0.3.26，PR #34）

- 并入 `8929669 → 5f6a41e`：
  - **v0.3.25 修我们报的 [BHXiang/ctpwa#5](https://github.com/BHXiang/ctpwa/issues/5)**：
    `computeMatrixErrors` 里 `B̄` 的维度原来按 `npartials` 取，`nSL > npart` 时越界读到 0，
    导致 `interference_rec_err` 整张为 0；改用 `n_amplitudes_`。
  - **v0.3.26**：`eigvalsh` 异常不再打断 `writeResult`；测试构建开 `-D_GLIBCXX_ASSERTIONS`；
    新增 `test_error_matrix_with_multi_ls_partials` 等回归。
- `setup.py`：并入上游的 `*_GLIBCXX_ASSERT_FLAGS`（CUDA `cxx` / CPU），保留 fork 的 `-std=c++20`，
  版本号随上游 `0.3.26`。

### `example/read_interference.py`

- 取上游新版：新增 `per_wave_efficiency()`，修**逐波效率**口径——矩阵元素是 `P_ii/W_rec`、
  `T_ii/W_gen`，必须先乘回 `W` 才是原始积分（旧公式漏了 `W_rec/W_gen` 因子）；
  并加一行说明。取后按本仓库约定 ruff-format。

### `example/README.md`

- §1：最低版本补 `≥ 0.3.25`（多 LS 分波下 `interference_rec_err` 正常）与 `≥ 0.3.26`
  （Hessian 病态时 `writeResult` 不被中断）；
- §10.4.2 / §10.4.4：每波效率明确 `P_ii = diag_rec × W_rec`、`T_ii = diag_gen × W_gen`（要乘回 `W`）；
- §10.4.3：`uproot` 依赖说明改为"`envs/ctpwa` 已装"。

### `example/README.md`

- §10.4.4 由「输出长什么样（真实例子）」改为**通用**的「输出块怎么读」：删掉来自具体分析
  的示例输出（样本量、分波名、`F_i`/`I_ij` 数值、`BF_i` 等），只保留各输出块的含义表；
- §10.4.1 里"本 config 的 `Number of partial waves: 17 (SL amplitudes: 42)`"改成通用写法
  （`N` / `M`，只说明"矩阵维度是分波数、不等于 SL 振幅数"）。
- 结论不变：`fit.py` 打印的 `fit fractions` 就是 `read_interference.py` 里那张逐波表。
- **README 只留"当前行为 + 用法"**，把"改了什么/以前怎样"统一放到本文件（`CHANGELOG.md`）：
  - §10.4 开头删掉 v0.3.23/v0.3.24 的实现沿革块，改为一句现状（重建侧 `B̄` 收缩、产生侧一趟
    `phsp_truth`、误差只含耦合）；
  - §1 路径口径、§7.3 正定判据、§7.7 warm-start、§9 GPU 策略、§11.5 标题里的"PR #/日期/旧版"表述全部去掉；
  - §8 标题去掉"（上游 `bc70a79` 并入）"，§11.5 改名"复现旧结果（`--amp-lambda`）"（只留操作步骤）；
  - §6/§10.4/§11.3 里"0.3.22 起取代旧的 `interference`"改为当前事实："没有 `interference` 对象，
    用 `interference_rec` / `interference_gen`"。

## 2026-11-24

### 上游同步（库；`example/` 上游未改动，merge 无冲突）

- （PR #33）并入上游 `cf4a23c → 8929669`（**v0.3.23 + v0.3.24**）：
  - v0.3.23：`writeResult` 的重建侧矩阵改用常驻 `B̄ = Σ(w/W)A A^H` 收缩（不读 `phsp`、不重算振幅；
    1M 事例 47.4 s → 24.3 s）；误差矩阵改**解析**传播（rec 由 `B̄` 代数、gen 复用一趟扫描的 ± 方向，
    成本与耦合数无关；200k/2 耦合 40 s → 1.9 s）；修 `reCalcAmp(θ)` 后 `B̄` 不重建；
    去掉 `[writeResult]`/`[phsp]` 进度打印，计时只在 `CTPWA_VERBOSE=1` 输出。
  - v0.3.24：修 0.3.23 的矩阵维度 bug（按逐 SL 振幅 → 按 `nSLvectors_` 合并 LS）；删除
    `getFitFractionMatrix` / `getFitFractionMatrixErrors` 公共 API（`tests/conftest.py` 改为
    "写 ROOT 再读回"复现）。已知限制：`interference_*_err` 只含耦合方向，θ 方向待解析 `dF` 路线。
- **`setup.py`：`-std=c++17` → `-std=c++20`**（CUDA `cxx` / `nvcc` / CPU 三处）。这是本 fork 第一处
  与上游不同的**非 `example/`** 改动——上游仍是 C++17，而 torch ≥ 2.13 的 C++ 头文件要求 C++20。

### `example/fit.py`

- **拟合分数输出改用 ROOT 里的相干口径**（与 `weight_best.root` 逐位一致）：
  - 默认从 `<output-dir>/weight_best.root` 读 `interference_gen`（产生侧）对角 → `F_i`，
    与 `read_interference.py` 的逐波表完全相同；`ΣF + ΣI = 1` 同时打印做闭合自检。
  - `σ` 取 `interference_gen_err` 对角（v0.3.23+ 只含耦合方向；Hessian 非正定时 C++ 不写，
    此时打一行警告、只给中心值）。
  - **不再计算/打印旧口径**：删掉 `compute_fit_fractions()`（非相干分母 `Σ=1`），
    日志只在来源行提示一句"旧日志里的 fit fractions 已停用，两者相差 `1/ΣF`"。
  - 读不到矩阵 / 没装 `uproot` → 只提示"不输出拟合分数"（**不再回退**旧的
    `getFitFractions`；对应地，"FF 失败 → 跳过效率"那条分支也一并删掉）。
  - 默认路径不再调 `getFitFractions`：省掉 `phsp_truth` 的一趟中心积分 + `2·2n` 趟扰动积分
    （本 config：82 个扰动方向 × 2）。
  - `optimization_summary.txt` 的 FF 段落带上口径说明（`ff_note`），无 σ 时写"± (无误差)"。

### `example/README.md`

- §1：最低版本补 `≥ 0.3.22`（ROOT 里的干涉矩阵 + `read_interference.py`）与 `≥ 0.3.24`
  （矩阵按分波合并 LS）；"本机当前"改为 0.3.24；新增"从源码编译用 C++20"一条。
- §10.4：标题去掉「（上游 v0.3.22）」；引言补 v0.3.23/0.3.24 的实现变化与维度 bug
  （"读到 0.3.23 生成的 ROOT 时矩阵会偏大"）；`*_err` 说明改为**只含耦合方向、θ 暂不传播**；
  "缺 `*_err`" 的原因改为"耦合自由块 Hessian 非正定"；扫描成本改为
  "rec 0 趟（常驻 `B̄`）+ gen 1 趟"，并说明误差矩阵成本与耦合数无关；
  计时段说明改为 `CTPWA_VERBOSE=1`（默认不再打印 `[writeResult] … ms`）。

## 2026-11-23

### 上游同步（库 + `example/fit.py`）

- （PR #30）并入上游 `5c1811a → cf4a23c`（**v0.3.22**，约 30 个提交）：
  - **FF 口径修复**：`writeResult` 写两张**按本样本相干总额归一**的对称矩阵
    `interference_rec`（重建侧）/ `interference_gen`（产生侧，`phsp_truth`）
    与参数误差矩阵 `interference_{rec,gen}_err`，另附 `integrals` 树
    （`N_rec/N_gen/W_rec/W_gen`）；`getFitFractions` 的"非相干分母"问题随之作废
    （HANDOFF §7.1 关闭，函数本身被标为待删）。
  - **θ 传播修复**：自由共振态参数现在真正进入 truth/efficiency 积分与 FF 矩阵
    （此前一律用配置初值，θ 导数恒为 0，参数误差无法传播）。
  - AmpGen：静态链因子显存预算 + 每 `(σ,事例,SL)` 物化一次、SL 复用、
    级联 BWR 闭式、缩并（lift）路径；64 位索引；新增
    `tests/{test_efficiency,test_matrix_errors,test_matrix_validation,test_static_budget,test_theta_propagation}.py`
    与 `scripts/cluster/multigpu_parity.py`。

### `example/fit.py`

- 6 处合并冲突按"两边并集"解决：
  - 保留 fork 的 7 个开关：`--cpu`、`--seed`（含 `FIT_SEED`）、`--cal-ff`、
    `--cal-eff`、`--ff-only`、`--err-mode`、`--err-tau`；
  - 采纳上游 `--polish-all` / `FIT_POLISH_ALL`：默认只抛 best，`--polish-all` 逐 run
    抛光，写 `polish_history.json`、`all_params/run_XX.pt`，`checkpoint.pt` 的
    `start_run` 顺延；抛光后按"正定优先、NLL 次之"重排（`_result_sort_key`）；
  - 抛光采纳规则与误差重算统一为"在**最终点**重算"（含未采纳分支），
    `_err_full` 始终与最终参数同步；
  - 摘要表增加 `polish` / `polishΔNLL` 两列（仅当有 run 被抛光时显示）；
  - `FitConfig` 新增 `polish_all: bool`（本地私有测试的字段集合已同步，测试不进仓库）。

### `example/README.md`

- 保留 fork 原有 §1–§11；新增 **§12 读取 `weight_best.root` 里的干涉矩阵**：
  两张矩阵的含义、闭合检验 `ΣF + ΣI = 1`、`read_interference.py` 用法，
  以及 `CTPWA_WR_NO_INTEGRALS` / `CTPWA_WR_NO_ERR` 开关。

### `example/read_interference.py`（新增）

- 取上游 v0.3.22 原文件：读 `interference_rec`/`interference_gen`（及 `*_err`、
  `integrals`），输出 `F_i ± σ`、n×n 表、闭合检验、非相干份额、平均/逐波效率，
  可选 `--bf-total` 折算分支比。

### 未动（本轮）

- `example/gen_phsp_mc.py`、`gen_toy_mc.py`、`.pre-commit-config.yaml` 为 fork 私有
  资产，保留不动（上游本次也未改动这三个文件）。

### 第二轮（PR #31：文档补全 + 去生产化）

#### `example/README.md`

- 目录补齐 §12（含 12.1–12.5 五个子节）；开头简介补上 §11/§12，并说明 §10 已不是最后一章。
- **§12 整节重写**（原来只有 5 段说明）：12.1 列出 `weight_best.root` 新增的 6 个对象
  （`interference_rec` / `interference_gen` / `*_err` / `integrals` / `legends`）+ 各自写入条件，
  并写明"矩阵维度 = **partial wave**（本 config 17×17），不是 `parameters.txt` 里的 42 个
  SL 耦合"；12.2 给出矩阵口径公式与「三个份额」对照表（`F_i` / `I_ij` / `FF_i` / `ε̄` /
  `ε_i` / `BF_i` / 原始积分）；12.3 用法 + 参数表 + 输出块顺序；12.4 **真实输出例子**
  （1/10 样本、`--runs 4 --niter 500`）逐项解读；12.5 常见问题 6 条。
- **修 §11.3 的过期片段**：旧文读 `weight_best.root["interference"]`，而 v0.3.22 起该对象
  不再写（→ `KeyError`）；改为 `interference_gen`、`d = 1 − 1/ΣF`，并注明 rec/gen 与旧矩阵的关系。
- §3.5 补第 4 条（默认只抛 best vs `--polish-all` 的取舍）；§4 Polish 表加 `--polish-all`；
  §5 加 `FIT_POLISH_ALL`；§6 加 `polish_history.json`（默认 polish 就有）与 `all_params/`，
  并把 `weight_best.root` 一行指向 §12；§10.1 输入直方图表把 `interference` 换成
  `interference_rec` / `interference_gen`。
- **去掉全部"生产环境"表述**（不再出现具体分析名/目录/路径）：§1 的遮蔽提醒、§3.4/§3.5 的
  数据集说明、§8.2 的默认值说明、§9 的作业目录说明、§10.2 的 `phsp_truth` 对比。

#### `example/fit.py`

- 删掉 `polish_one_result` 里多余的 `optimizer._hessian_fields(rep2)`：本分支的优化器没有
  这个方法（上游 API 名不同），运行时只会打一行"抛光后 Hessian 子空间诊断失败"warning；
  子空间字段本来就由随后的 `compute_param_errors` + `_err_fields` 在**最终点**重算。
  删后 warning 消失，`pytest example/tests -q` → 55 passed。

#### `example/gen_phsp_mc.py`

- 文件头与 `--help` epilog：「不等于生产 `phsp_truth`」→「不等于带效率的 `phsp_truth`」。

#### 验证（本轮）

- `--runs 4 --niter 500 --seed 42`（1/10 样本、1×V100）：8:51、峰值 1.7 GB；polish 前
  −5767.089 → polish 后 **−5851.279**；`weight_best.root` 闭合 `ΣF+ΣI = 1.000000`、
  `ΣF = 2.129`、`ε̄ = 1.003`；`*_err` 因 Hessian 非正定未写（符合预期）。
- `--runs 2 --niter 100 --polish-all`：两条记录都 accepted；`all_params/run_XX.pt`、
  `polish_history.json`、`checkpoint.pt`（`start_run=2`）、`best_params.pt`（= best run 逐位相同）、
  摘要表 `polish`/`polishΔNLL` 两列均正常；无 `Traceback`/`ERROR`。

### 第三轮（PR #32：章节归位）

- `example/README.md`：把上一轮新增的 §12 移进 **§10.4**——按本文自己的分类原则，
  `read_interference.py` 和 `plot.py`、两个 MC 脚本一样是"不属于 `fit.py` 用法"的脚本，
  应归入 §10「其它脚本」。子节编号改为 **10.4.1–10.4.5**（与 §10.1 的 `####` 层级一致）；
  §6 输出文件、§10.1 输入直方图表、§11.3 干涉深度三处指针改为「见 §10.4」；
  §10 引言由"本章三个脚本"改为"本章四个脚本"，并点名 10.1 画图 / 10.2–10.3 生成 MC / 10.4 读矩阵。

## 2026-11-12

### 上游同步（不涉及 `example/`）

- 2026-11-13（PR #26）并入上游 `e3cf387 → 267254f`（**v0.3.19**，2 个提交）：干涉矩阵宿主累加器
  未初始化（`new double[n*n]` → `new double[n*n]()`，NaN/1e170）+ 非对角干涉 `float→double`，
  新增 `tests/test_interference.py`；发布相关（版本号、根 `CHANGELOG.md`/`README.en.md`、`MANIFEST.in`）。

### `example/fit.py`

- 2026-11-13（PR #27）**`--amp-lambda` 默认值 `1e-4` → `0`（默认关闭幅度罚项）**：
  - 改动点：`UnifiedPWAOptimizer.__init__` 的 env 兜底 `FIT_AMP_LAMBDA` 默认 `"1e-4"` → `"0"`；
    `resolve_args` 的 `_env_float("FIT_AMP_LAMBDA", 1e-4)` → `0.0`；argparse 帮助文本同步。
  - 启动 dump 改为 `lambda=0(off)` / `lambda=0.001`，一眼可见本作业开没开。
  - 逻辑无需改：两处罚项（主循环 `1147`、降维重拟合 `2094`）都已有 `> 0` 守卫。
  - ⚠ **与上游分歧**：上游默认 `1e-4`。
  - 前向数值影响可忽略：`λ` 等价于 σ=1/√(2λ) 的高斯先验（`1e-4` → σ≈70.7，以参考波 `|v_0|=1`
    为单位），对现实干涉深度（`d=0.1–0.95`）远小于“开始咬人”的阈值（`λ ≳ 3e-2`）。推导见
    README §11.3。
  - ⚠ **复现旧结果**需显式 `--amp-lambda 1e-4` / `FIT_AMP_LAMBDA=1e-4`。

### `example/tests/test_errors.py`

- 2026-11-13（PR #27）`test_default_optimizer_and_err_mode_policy`：`amp_lambda` 默认断言
  `1e-4` → `0.0`；并新增「显式 `--amp-lambda 1e-3` 生效」的断言。

### `example/README.md`

- 2026-11-13（PR #27）**新增 §11「干涉与幅度控制三参数」**（11.1 干涉简并方向 / 11.2 三者分工 /
  11.3 由预期干涉强度反推 `--amp-lambda` / 11.4 推荐流程 / 11.5 默认值变更与复现），含
  `d = 1 − T/D`、`a(d) = (1−√(1−d²))/d`、`λ ≤ 1/(2k²a²)` 与数值表；并从 `weight_best.root`
  的 `interference` 矩阵取 `d` 的代码片段（含口径说明：对角**无**因子 2，与内部 `scattering`
  不是同一套）。
- 2026-11-13（PR #27）同步更新 `--amp-lambda` 默认值的所有出现：§0 TL;DR、§3.1、§3.4、§4 参数总表、
  §5 环境变量示例、§7 备注；并修正 §10.3 末尾「`fit.py` 仍硬要求 CUDA」的过期表述（PR #24 起有 `--cpu`）。

---

## 2026-10-30

### `example/fit.py`

- 2026-10-30（PR #24）新增 **`--cpu`**：**默认仍要求 GPU**——后端不是 `cuda` 时**直接报错退出**
  （提示可加 `--cpu`）；只有显式要 CPU 才继续：
  - `--cpu`（或在外面显式 `CTPWA_DEVICE=cpu`）→ 用 CPU 后端拟合，打印「比 GPU 慢 20–60 倍，仅适合
    小样本/调试」警告；`--cpu` 会**覆盖**已有 `CTPWA_DEVICE`（覆盖时打印一行提示）。
  - 实现要点：① `--cpu` 在 **`import ctpwa` 之前**从 argv 取——ctpwa（≥0.3.15）在 import 期就按
    `CTPWA_DEVICE` **互斥加载** `_cuda`/`_cpu`，import 后再换来不及；② 新增模块级 `_DEV`
    （由 `ctpwa.backend()` 推导、带 stub 兜底），把原先写死的 3 处 `device="cuda"`
    （`_generate_initial_params` / 兼容包装器 / `UnifiedPWAOptimizer`）改为 `_DEV`——否则 CPU 后端下
    会把 CUDA 张量喂给 `_cpu.so`；③ P4.1 由「无 CUDA 直接退出」改为「**非显式 CPU 才退出**」。
  - ⚠ **与上游分歧**：上游 `example/fit.py` 硬要求 CUDA、无此开关；将来合并上游时注意。
  - GPU 路径行为不变（`test14` T0–T8 + golden 回归通过）；CPU 回退另有 `test17`。

### `example/README.md`

- 2026-10-30（PR #24）§1 运行前提补 **CPU 回退（`--cpu`）** 一条；§4 参数总表加 `--cpu` 行；
  §9 后端选择加「`fit.py` 自己的策略（默认要求 GPU，`--cpu`/`CTPWA_DEVICE=cpu` 才用 CPU）」小节。

---

## 2026-10-29

### `example/fit.py`

#### 参数误差 / Hessian

- 2026-10-29（PR #19，并入上游 `fa40d41`）**正定性改为“可行子空间”判据（KKT 二阶条件）**：
  `_errors_from_hessian` 内部改用 `ctpwa.diagnostics.analyze_hessian`——
  剔固定参考（Re_0/Im_0）→ 剔**活跃**参数（贴边且梯度朝外、不计入判别且误差标 NaN）→
  在剩余**可行子空间**上判正定/求误差。`is_positive_definite` = `reduced_pd`，
  新增 `is_positive_definite_full`（旧口径全自由块）、`n_active`/`active_params`/
  `full_min_eigenvalue`/`kkt_grad_max`/`kkt_ok`/`n_negative_*` 字段；
  误差按 grouped `[Re.., Im.., θ..]` 用 `map_errors_to_params` 映射（不再交错取 `sd[2i]`）。
  保留原有 `auto/strict/pinv/psd` + 有效秩截断 + 协方差 CSV 语义不变。
- 2026-10-29（PR #19）终态 `final_nll` 与**梯度**同在 `final_params` 上求（梯度供活跃集判别）；
  polish 采纳规则放宽为 `nll2 ≤ old+1e-6 或 (pd2 and not old_pd)`，采纳/未采纳都在**最终点**
  重算误差并把子空间字段同步回 `best_res`（摘要/CSV 与最终参数始终一致）。
- 2026-10-29（PR #19）`extract_coupling_complex` dtype 判定改为
  `build_info()['complex_bytes']` 优先、`DeviceManager().compiledPrecision()` 探针兜底、
  再按 params 精度兜底；并删掉一处重复定义的旧 `extract_coupling_complex`（PR #18 合并残留死代码）。

#### 参数削减 / 边界诊断

- 2026-10-29（PR #19）新增**后端自检**：header 打印
  `backend=… precision=… complex_bytes=… has_cuda=…`（`ctpwa.build_info()`），
  非 `cuda` 时 `log.warning`（新门面 0.3.15+ 会在 GPU 不可用时**静默退 CPU**）。
  摘要增 `正定判据=可行子空间(KKT)`，run 表列由 `正定` 改为 `正定(可行)` + 新增 `贴边` 列；
  单 run 与总表打印改 `正定(active) = … [full=…]`。`_hessian_fields` 合并进
  `_err_fields`（err dict → 字段的唯一出口）；`compute_reduction` 的活跃集改用
  `_hessian_diagnosis`（与误差/抛光同一条可行子空间规则）。

#### 路径口径

- 2026-10-29（PR #21）**统一路径锚点**，`fit.py` 三条改动：
  1. **import 期 chdir 到 config 目录**（在构造 `ctpwa.analysis()` **之前**）：C++ 在构造期
     就按 **cwd** 打开 `data`/`phsp`/`bkg`，而 `phsp_truth` 是 FF/效率时才打开、
     `Resonances`/`Constraints.chains(_exact)` 由 C++ 按 config 目录解析（`src/Config.cu`
     的 `parent_path()`）→ 提前 chdir 后三类口径统一为「config 内相对路径 = 相对 config 目录」。
  2. 新增 `_absolutize_io_paths(cfg, base=_ORIG_CWD)`：把 `config`/`output_dir`/`warm_start_path`
     （含 `--warm-start auto` 与 `FIT_WARM` 派生的那条）固化为**绝对路径**，锚在**调用者原始
     cwd**（`resolve_args()` 跑的时候 cwd 已经变成 config 目录，所以不能直接用
     `os.path.abspath`）。
  3. 新增 `_config_data_paths()` + 启动 dump：打印 config 里 `data/phsp/phsp_truth/bkg/bkg_weights`
     的**解析结果**（绝对路径 + 是否存在；缺失标 `← 不存在!`）；只做文本粗读，不做 YAML 解析。
  - **行为变化**：`--output-dir`/`--warm-start`/`--resume` 由「相对 config 目录」（反直觉、会把
    结果写到 config 目录下）改为「相对调用目录」；`data/phsp/bkg` 的相对路径由「相对启动目录」
    改为「相对 config 目录」（与 `phsp_truth` 一致）。**cwd == config 目录时（最常见用法）
    全部是 no-op。**
  - 新增 `_ORIG_CWD`（记录 chdir 前的 cwd）；`_config_path_from_argv` 的 docstring 同步更正
    （原文那句"只影响 chdir"已过期）。
  - 本地单测 `example/tests/test_paths.py`（8 项，CPU）；GPU smoke 见 `../test16/`。
  - ⚠ 与上游分歧：上游 1fda180 仍是「main() 里 chdir」+ Data 段路径原样存字符串（`src/Config.cu`
    `parseData` 不做 `parent_path()`），因此上游没有"config 内相对路径统一"这一口径；将来合并
    上游时注意这一点。

### `example/README.md`

- 2026-10-29（PR #19）新增 **§9 后端选择（CPU/GPU）与 `CTPWA_DEVICE`**；§1/§5 补 ctpwa
  版本要求与新门面环境变量；§7 备注补“正定判据变为可行子空间(KKT)”与“仓库根 `ctpwa/`
  会遮蔽已装 wheel”两条坑；§8 说明误差口径已改为可行子空间。
- 2026-10-29（PR #20）新增 **§10 MC 生成（`gen_phsp_mc.py` / `gen_toy_mc.py`）**。
- 2026-10-29（PR #22）**章节重组**：把**不属于 `fit.py` 用法**的脚本归入最后一章
  **§10「其它脚本」**（10.1 `plot.py` / 10.2 `gen_phsp_mc.py` / 10.3 `gen_toy_mc.py`）；
  原 §9/§10/§11 顺次重编为 **§8/§9/§10**，一级标题改为 `example 使用说明（fit.py 为主）`；
  开头加**两级目录（TOC，33 条）**。**10.2 新增参数表**、**10.3 表补全**为三列
  （参数/说明/默认），并补充：`.dat` 每行固定 4 个数、**无权重列**（权重走 `root`/`npy` 或
  config 的 `data_weights`/`bkg_weights`/`phsp_weights` 独立权重文件），以及
  `--mode unweighted`（平坦 + `|M|²` 舍选 AR → 等权，`stats.acceptance`/`max_weight`）与
  `weighted`（全收 + 权重 → 快，需配 `--poisson` 才得到等权 `dat`）的语义；另补
  **`--seed` 默认取当前 Unix 时间戳（与 `fit.py` 一致、每次运行不同）** 和
  **`--check-distribution` 自洽性检查的详解**（独立加权平坦参照样本 → `m²(i,j)` 归一化直方图
  `max|Δ|<0.05`、统计涨落、等权前提、与闭包的区别）；并修掉**列表里多余的反斜杠转义**
  （`|M|²`/`max|Δ|` 等——转义只在**表格单元格**里需要，列表/正文里会原样显示反斜杠）。

### `example/gen_phsp_mc.py`

- 2026-10-29（PR #20）**DIYMC P1 平坦相空间生成**（自包含单文件，不 import 本仓库其它文件）：
  `--config -n -o [--seed --format dat|root|npy --mother --mother-mass --weighted --chunk
  --force --no-check --check-events]`；`dat` 输出 = 每事件 n_final 行 `E px py pz`（12 位小数），
  末态顺序 = `Data.order`；内置自检（末态顺序 vs `Data.order`、守恒、质量壳、行数 = n×n_final、
  `weightMax`）；`--chunk` 分块追加写以支持千万级样本。**不需要 GPU，也不读数据文件**。
  文档里写清两条语义：平坦 PS ≠ 带效率的 `phsp_truth`；closure 用途需百万级样本（窄共振）。
- 2026-10-29（PR #22）`--seed` 默认由 `0` 改为**当前 Unix 时间戳**（与 `fit.py` 一致）：
  每次运行都不同，避免"忘了改 seed"导致重复样本；启动打印 `[seed] … 复现本作业请加 --seed N`，
  分块时第 k 块仍用 `seed+k`。要复现就显式传 `--seed`。

### `example/gen_toy_mc.py`

- 2026-10-29（PR #20）**DIYMC P2/P3 振幅加权 toy → pseudo-data**（自包含单文件）：
  `--config --params-from(best_params.pt) -n -o [--seed --mode unweighted|weighted --batch
  --max-trials --format --poisson --nobs|--nobs-from-data --allow-short --bkg/--no-bkg
  --bkg-scale --no-shuffle --write-config --stats-json --check-distribution --check-events
  --check-pair --check-bins]`；`--poisson` 抽成等权 pseudo-data（等权无放回 / 带权按 w 有放回）；
  `--bkg` 按 `bkg_weights` 加权抽样叠加真本底；`--write-config` 派生 config（把 `data` 指向 toy）；
  `--check-distribution` 用 `m²(i,j)` 分布对照 `|M|²` 加权样本做自洽性检查。所有便宜的错误路径
  （参数文件/长度、weighted+dat 组合）都在构造 `analysis` **之前**报错。
  参数按当前后端自动搬到设备：**CUDA 后端要求 `params` 是 CUDA 张量**，而 `best_params.pt`
  是 fit.py `.cpu()` 存的（首轮 GPU 冒烟就是踩了这个 `TORCH_CHECK(ctpwa_is_cuda_like(params))`）。
- 2026-10-29（PR #20）`--format root` 写盘前把事件/权重转成 `torch.Tensor`（新工具
  `as_torch()`）：`analysis.writeMC()` 的第一个形参是 `torch::Tensor`，pybind11 **不做**
  numpy→torch 自动转换，脚本内部又是 numpy 数组 → 第四轮 GPU 冒烟抛
  `TypeError: writeMC(): incompatible function arguments`（`gen_phsp_mc.py` 走
  `gen.write(ev, …)`，`ev` 本来就是 torch 张量，故未踩到）。
- 2026-10-29（PR #22）`--seed` 默认由 `0` 改为**当前 Unix 时间戳**（与 `fit.py` 一致）：
  每次运行都不同（避免"忘了改 seed"导致重复 toy）；启动打印 `[seed] … 复现本作业请加
  --seed N`，`--stats-json` 里的 `seed` 也记录实际值。`--check-distribution` 参照样本仍用
  `seed+9999`、本地 RNG 用 `seed+20260101`。

### `example/CHANGELOG.md`

- 2026-10-29（PR #22）**本文件重组**：由「按文件」改为 **按合并日期（`##`）→ 文件（`###`）**；
  不涉及 `example/` 的上游同步按日期归入各日期的「上游同步」小节；`fit.py` 的变更类别
  保留为 `####` 小类。历史条目里对 `example/README.md` 的 `§` 引用同步改为重组后的编号。

### 上游同步

- 2026-10-29（PR #19）并入上游 `BHXiang/ctpwa`（`0e452a1` → `1fda180`，12 个提交，
  tag v0.3.15–v0.3.18）：
  - `039d7df`/`c06b4fc` DIYMC 相空间产生器（native GENBOD + 振幅加权；新增
    `analysis.getIntensity/generateMC/writeMC`、`PhaseSpaceGenerator`）——**不影响拟合流程**；
  - `ce9d993`/`f202167`（v0.3.15，breaking）**单 wheel 双后端**：新增 `ctpwa/` 门面包
    （`__init__.py`：import 时探测后端、GPU 优先；`CTPWA_DEVICE=auto|cuda|cpu`；`build_info()`/
    `backend()`/`BACKEND`/`FALLBACK_REASON`）、`include/Backend.h`、`CudaCompat.h`、
    `include/cpu_shims/*`、`src/Backend.cpp`、`src/JitStubs_cpu.cpp`、`scripts/port_cuda_to_cpu.py`、
    `setup.py` 重写（含 CPU 构建路径）；
  - `fdd328c`（v0.3.16）恢复被门面裁掉的公开 API；
  - `f82315a` `phsp_weights` 进入 NLL 归一化/梯度/Hessian（当前 config 未用
    `phsp_weights` → 现有结果不受影响）；
  - `ecaa513` 修 bkg `g_v` 在耦合变换二阶项中的符号（**会改变带本底拟合的 Hessian/误差**，
    中心值不变）；
  - `fa40d41`（v0.3.17）新增 `ctpwa/diagnostics.py`（`analyze_hessian` 可行子空间判据 +
    `map_errors_to_params` grouped 映射）并改写 `example/fit.py` 的正定/误差路径；
  - `1fda180` `example/fit.py` 耦合 dtype 用 `build_info()`、秩门槛打印带尺度；
  - `1fa9f23` 仓库与 sdist 不再发布 `doc/`（我们本来没有）；
  - 涉及文件：`ctpwa/*`、`include/*`、`src/*`、`setup.py`、`tests/*`（新增 `test_cpu_*`、
    `test_mc*`、`test_hessian_diagnostics.py`）、`example/fit.py`（唯一冲突文件，7 处）。
  - 环境：`ctpwa` pip 需 ≥ 0.3.17（`ctpwa.diagnostics`）；本机 0.3.18。`CTPWA_DEVICE=cuda`
    已在 `test14/sub.gpu` 等作业脚本中显式设置（避免静默退 CPU）。

---

## 2026-10-22

### `example/fit.py`

#### 参数误差 / Hessian

- 2026-10-22（PR #18，并入上游 `bc70a79`）**非正定时改走有效秩截断**（新开关
  `--no-errors-truncate` 可退回旧的 pinv）：`--err-mode auto` 且 Hessian 不定时，保留
  `λ_i > τ·λmax` 的方向做误差传播 `σ²_m = Σ_{i∈keep} E[m,i]²/λ_i`，**每个参数仍有有限误差**，
  被冻结的线性组合（未测定方向）单独列出并记录在 `_last_error_truncation`；
  `strict/pinv/psd` 显式指定时语义不变。非正定警告措辞同步更新。

#### 参数削减 / 边界诊断（并入上游 `bc70a79`）

- 2026-10-22（PR #18）新增 `--fit-mode {0,1}`：`0`=FREEPARAMS chain×step（默认）、
  `1`=VSPACE 逐振幅（Hessian 二阶项恒为 0 → 精确）。fit_mode 只能在 `ctpwa.analysis()`
  构造期决定，故在 **import 期**用 `_fit_mode_from_argv()` 从 argv/`FIT_MODE` 取出并传给
  `ctpwa.analysis(cfg, _FIT_MODE)`（需 ctpwa ≥ 0.3.10）。
- 2026-10-22（PR #18）新增 `--reduce-params`（配合 `--rank-tau`/`--rank-iters`/
  `--rank-fixed-point`/`--rank-scale`）：用精确 Hessian 的**有效秩**构造降维基
  `p = p0 + E·y`，在保留子空间做降维重拟合，输出冻结组合与全参数误差到
  `param_reduction.txt` / `best_params_reduced.pt`；采用降维解时会打印并置
  `_err_full=None`（协方差/相关矩阵 CSV 在最终点重算）。
- 2026-10-22（PR #18）新增 `--bound-margin`（默认 0.02）：列出贴住 `free_range` 的 θ
  （边距 + `|∂NLL/∂θ|`），标注“边界/非驻点评估 → 假非正定”。
- 摘要 `optimization_summary.txt` 增 `fit_mode` / `err_mode`+截断 / 参数削减 `k` 三行。

### `example/README.md`

- 2026-10-22（PR #18）新增 **§8 参数削减 / 误差截断 / 边界诊断 / `--fit-mode`** 参数表与
  使用建议（含 `--reduce-params` 会替换最终解、默认不开的说明）。

### 上游同步

- 2026-10-22（PR #18）并入上游 `BHXiang/ctpwa`（`68e0d0c` → `0e452a1`，13 个提交，
  tag v0.3.10–v0.3.14）：
  - `77b2bd6` trans 约束名可为链名/子链片段；配对改最小差异匹配（v0.3.10）；
  - `98ed8f2`/`970c77b` phsp Hessian 大 n 回退（n² 共享内存超限）+ `CTPWA_PHSP_HESS_DEBUG`（v0.3.11）；
  - `63e8206`/`7b2cd2c` identical 每-σ 四动量缓冲被提前释放修复 + ident4 回归（v0.3.12）；
  - `c997a3f` Hessian 乘法链二阶项用 `v_a/(p_j·p_k)`、补齐 step×step 项（v0.3.13）；
  - `50259f2` Info.cu 同一 intermediate 下重复 `[J,P]` 块被覆盖 → 共振态丢失 + 重复列两次（v0.3.14）；
  - `bc70a79` `example/fit.py` 新功能（见上两节），唯一与我们的冲突点，按“双方都保留”手工合并；
  - 涉及文件：`include/AmpGen.cuh`、`src/{AmpGen,ComputeHessian,Info,Parameters}.cu`、
    `setup.py`、`tests/*`（新增 ident4 / trans_multi_step / dupjp / hess_mult 等回归）、`example/fit.py`。
  - 环境：`ctpwa` pip 包需 ≥ 0.3.10（`analysis(config, fit_mode)`）；本机 0.3.13
    （含全部 v0.3.13 C++）。`50259f2` 只在 config 存在“重复 `[J,P]` 块”时才影响结果；
    当前配置无此结构 → 现有结果不受影响。

---

## 2026-09-29

### `example/plot.py`

- 2026-09-29（PR #15 `1f66ceb`）**Dalitz(2D) pull/Fit 加入本底**：`total_fit_values = hfit+hbkg`；
  Pull 用 `(data-total)/√total`；Fit 面板画 total、标题有本底时 `Fit+Bkg`；`vmax=max(data,total)`。
  与 1D（`plot_combined_histogram_with_pull`）口径一致。
- 2026-09-29（PR #17，https://github.com/wwdws1/ctpwa/pull/17 ）修 `--params N` 路径：
  原先调用不存在的 `analysis.writeWeightFile`，且参数文件名/解析都不对 → 改为从
  `results/parameters.txt` 按 `# RUN: run_N` 分块正确解析（耦合 `±`/`(fixed)` 与共振态行），
  拼 `[Re, Im, θ]`（float64）后调 `analysis.writeResult`；失败**明确报错**、不再静默回退。

### `example/README.md`

- 2026-09-29（PR #16）末尾新增 **§10.1 `plot.py` 使用说明**。
- 2026-09-29（PR #17）开头注明 `plot.py` 见 §10.1、变更历史见 `CHANGELOG.md`。

### 上游同步

- 2026-09-29（PR #16）并入上游 `BHXiang/ctpwa`（`a3946d5` → `68e0d0c`），**不涉及 `example/`**：
  - `68e0d0c` 修大 `phsp_truth` 下 `getFitFractions`/`getEfficiency` 崩溃与内存泄漏（issue #3）；
  - `7c59d36` trans 支持 N 个链名 + 每名字一个比值；ONE 占位质量不进振幅；
  - `754cd98` 发布 v0.3.9（`setup.py`）。
  - 涉及文件：`include/*`、`src/*`、`tests/configs/trans_multi*.yml`、`tests/test_trans_host.py`、`setup.py`。

---

## 2026-09-25

### `example/fit.py`

#### FF（拟合分数）/ 分波效率

- 2026-09-25（PR #9 `4d24aa6`）FF/效率改为开关：`--cal-ff {True,False}`（默认 True）、
  `--cal-eff {True,False}`（默认 False）；摘要标注“未计算”。

#### 日志

- 2026-09-25（PR #10 `5a5dc24`）参数误差日志标签 `[errors]` → `[param-err]`（避免误读为报错）。
- 2026-09-25（PR #13 `7765e0d`）输出/日志约定：内部诊断（pLBFGS/polish）由 `print` 改 `log`，
  结果表与可复现 tag 保留 `print`。

#### 随机初值 / 输出文件

- 2026-09-25（PR #12 `279e5e8`）随机初值 `--seed <int>` / `FIT_SEED`（base 默认当前 Unix
  时间戳，run i 用 `base+i`；`--seed 42` 复现旧行为；启动打印可复现命令；`checkpoint.pt` 存 seed，
  `--resume` 以 checkpoint 为准）；best 点输出 `param_covariance.csv` / `param_correlation.csv`
  （完整 N×N，仅正常拟合、仅 best；log 只提示）。

#### 代码整理

- 2026-09-25（PR #13）常量上移、删死代码/历史叙事、`_project_params_`→`_project_params`、
  去重 `generate_initial_params`（bit-exact）、补类型注解 + `ErrDict/RunResult`、
  `cfg` dict → `FitConfig(Mapping)` 兼容式 dataclass、`main()` 抽纯函数
  `_determine_base_seed/_seed_message`；**修 `--warm-start auto` 按 `--output-dir` 找
  `best_params.pt`**（旧版恒指 `results/`）。
- 2026-09-25（PR #14）polish 终态 verbose 在“全活跃”时的 `lmax` NameError；非 PD 且
  `err_mode=strict` 时的警告措辞；参数矩阵输出复用 best 误差（去掉重复 `[param-err]` 与重复 eigh）。

#### 文档引用清理

- 2026-09-25（PR #11 `4b6b3a9`）去掉注释中指向仓库外/不存在的文档引用，以及内部编号标注。

### `example/README.md`

- 2026-09-25（PR #9 `4d24aa6`）`--cal-ff`/`--cal-eff` 参数表与说明。
- 2026-09-25（PR #12 `279e5e8`）`--seed`/`FIT_SEED`、输出文件表加参数矩阵、seed 说明。
- 2026-09-25（PR #13 `2d9c761`）`--warm-start auto` 目录修正 + 输出/日志约定。

### `example/.pre-commit-config.yaml`

- 2026-09-25（PR #11 `4b6b3a9`）清理注释中指向仓库外文档的引用。

---

## 2026-09-24

### `example/fit.py`

#### FF（拟合分数）/ 分波效率

- 2026-09-24（PR #6）放开 FF/效率：非 PD 时按 `err_mode` 用 PSD 化曲率传播误差、**去掉 PD 门控**
  （中心值恒输出）；`extract_coupling_complex` 用 `ctpwa.DeviceManager().compiledPrecision()`
  选复数 dtype；FF/效率放到最后、核心摘要先写、新增 `--ff-only`。
- 2026-09-24（PR #7）FF 崩溃不再损坏核心摘要；`--ff-only` 输出隔离到 `<output-dir>/ff_only/`。

#### 日志

- 2026-09-24（PR #8 `9a88ebc`）`logging.basicConfig(stream=sys.stdout)`：日志进 `.log`。

### `example/plot.py`

- 2026-09-24（PR #7 `b1924f4`）`writeWeightFile` 的复数 dtype 按 `.so` 编译精度选择
  （`complex128`/`complex64`），修 “vector dtype must match .so complex precision”。

### `example/README.md`

- 2026-09-24（PR #6 `e4e0ff3`）FF/效率不再 PD 门控 + 大 `phsp_truth` 崩溃提示 + `--ff-only`。

---

## 2026-09-23

### `example/fit.py`

#### 可观测性 / 方法学

- 2026-09-23（PR #4）去掉静默降级（polish/FF/保存失败改 `log.exception` + `polish_status`）；
  reparam profile（`FIT_OPT_PROF`）；`optimization_summary.txt` 增 `eval数/n_iter` 列；
  all-runs 系综统计（best/median/IQR/std/worst）；`FIT_DETERMINISTIC`；等价微优化
  （`_reparam_unpack` 预分配、polish 免 n×n 分配、初始化向量化，数值逐位一致）。

### `example/README.md`

- 2026-09-23（PR #5 `43ac312`）修过期内容：默认优化器措辞、`--config` 已生效、环境变量补全、
  摘要列/系综、§3.5 来源标注。

---

## 2026-09-22

### `example/fit.py`

#### 优化器

- 2026-09-22（PR #1 `b7865bc`）新增 `reparam` 有界优化器并设为默认：S1 幅度软墙
  `amp=amp_max·sigmoid(u)`、S2 幅度罚项 `λ·Σ|A|²`、S4 polish 加固（步长夹/硬帽/噪声底停机）。
- 2026-09-22（并入上游 `4e5e5e5`，含上游 `a3946d5`）上游采纳我方 reparam；并入 projected
  闭式 two-loop + `FIT_OPT_PROF` 诊断、`--config` 修复、`reparam-tol-stop` 更名 +
  polish `cand_best is None` 保护。
- 2026-09-22（PR #3 `4a29b3f`）恢复上游默认 `optimizer=projected`；`reparam` 改为可选/推荐；
  `err_mode` 默认随优化器（reparam→auto，其它→strict）。

#### 参数误差 / Hessian

- 2026-09-22（PR #2 `41c2de0`）Hessian 非正定时的参数误差回退：
  `--err-mode {auto,strict,pinv,psd}` + `--err-tau`（默认策略 C：显式 > `FIT_ERR_MODE` >
  优化器相关）。
- 2026-09-22（PR #2 `3ea973c`）`final_nll` 取**最终接受点**真实 NLL；`parameters.txt` 第 0 行
  固定参考统一标 `(fixed)`；`getHessian` 耦合块顺序经有限差分确认为 **BLOCK**。

### `example/plot.py`

- 2026-09-22（PR #2 `af0e72b`）应用 ruff-format。

### `example/README.md`

- 2026-09-22（PR #1/#2/#3）随 reparam / err-mode / 默认优化器更新。

### `example/.pre-commit-config.yaml`

- 2026-09-22（PR #2 `fdbf26f`/`af0e72b`）新增：仅作用于 `example/`；含 ruff-format 等会改文件的
  hook；`ruff` 仅 `--select E9,F63,F7,F82`。
