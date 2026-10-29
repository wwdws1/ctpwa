# CHANGELOG（`example/`）

本文件记录对上游 `BHXiang/ctpwa` 的改动中、**`example/` 目录内**各文件的内容变化
（PR #1–#21）。其它目录随上游维护，不在本表；上游同步见文末。
日期 = 合并进 `main` 的日期；“PR #n”指 `wwdws1/ctpwa` 的合并请求。

> 只列**行为/接口/文档**变化；纯格式（ruff-format/空白）只在首次记录。

---

## `example/fit.py`

### 优化器
- 2026-09-22（PR #1 `b7865bc`）新增 `reparam` 有界优化器并设为默认：S1 幅度软墙
  `amp=amp_max·sigmoid(u)`、S2 幅度罚项 `λ·Σ|A|²`、S4 polish 加固（步长夹/硬帽/噪声底停机）。
- 2026-09-22（并入上游 `4e5e5e5`，含上游 `a3946d5`）上游采纳我方 reparam；并入 projected
  闭式 two-loop + `FIT_OPT_PROF` 诊断、`--config` 修复、`reparam-tol-stop` 更名 +
  polish `cand_best is None` 保护。
- 2026-09-22（PR #3 `4a29b3f`）恢复上游默认 `optimizer=projected`；`reparam` 改为可选/推荐；
  `err_mode` 默认随优化器（reparam→auto，其它→strict）。

### 参数误差 / Hessian
- 2026-09-22（PR #2 `41c2de0`）Hessian 非正定时的参数误差回退：
  `--err-mode {auto,strict,pinv,psd}` + `--err-tau`（默认策略 C：显式 > `FIT_ERR_MODE` >
  优化器相关）。
- 2026-09-22（PR #2 `3ea973c`）`final_nll` 取**最终接受点**真实 NLL；`parameters.txt` 第 0 行
  固定参考统一标 `(fixed)`；`getHessian` 耦合块顺序经有限差分确认为 **BLOCK**。
- 2026-10-22（PR #18，并入上游 `bc70a79`）**非正定时改走有效秩截断**（新开关
  `--no-errors-truncate` 可退回旧的 pinv）：`--err-mode auto` 且 Hessian 不定时，保留
  `λ_i > τ·λmax` 的方向做误差传播 `σ²_m = Σ_{i∈keep} E[m,i]²/λ_i`，**每个参数仍有有限误差**，
  被冻结的线性组合（未测定方向）单独列出并记录在 `_last_error_truncation`；
  `strict/pinv/psd` 显式指定时语义不变。非正定警告措辞同步更新。
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

### 参数削减 / 边界诊断（并入上游 `bc70a79`）
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
- 2026-10-29（PR #19）新增**后端自检**：header 打印
  `backend=… precision=… complex_bytes=… has_cuda=…`（`ctpwa.build_info()`），
  非 `cuda` 时 `log.warning`（新门面 0.3.15+ 会在 GPU 不可用时**静默退 CPU**）。
  摘要增 `正定判据=可行子空间(KKT)`，run 表列由 `正定` 改为 `正定(可行)` + 新增 `贴边` 列；
  单 run 与总表打印改 `正定(active) = … [full=…]`。`_hessian_fields` 合并进
  `_err_fields`（err dict → 字段的唯一出口）；`compute_reduction` 的活跃集改用
  `_hessian_diagnosis`（与误差/抛光同一条可行子空间规则）。

### 可观测性 / 方法学
- 2026-09-23（PR #4）去掉静默降级（polish/FF/保存失败改 `log.exception` + `polish_status`）；
  reparam profile（`FIT_OPT_PROF`）；`optimization_summary.txt` 增 `eval数/n_iter` 列；
  all-runs 系综统计（best/median/IQR/std/worst）；`FIT_DETERMINISTIC`；等价微优化
  （`_reparam_unpack` 预分配、polish 免 n×n 分配、初始化向量化，数值逐位一致）。

### FF（拟合分数）/ 分波效率
- 2026-09-24（PR #6）放开 FF/效率：非 PD 时按 `err_mode` 用 PSD 化曲率传播误差、**去掉 PD 门控**
  （中心值恒输出）；`extract_coupling_complex` 用 `ctpwa.DeviceManager().compiledPrecision()`
  选复数 dtype；FF/效率放到最后、核心摘要先写、新增 `--ff-only`。
- 2026-09-24（PR #7）FF 崩溃不再损坏核心摘要；`--ff-only` 输出隔离到 `<output-dir>/ff_only/`。
- 2026-09-25（PR #9 `4d24aa6`）FF/效率改为开关：`--cal-ff {True,False}`（默认 True）、
  `--cal-eff {True,False}`（默认 False）；摘要标注“未计算”。

### 日志
- 2026-09-24（PR #8 `9a88ebc`）`logging.basicConfig(stream=sys.stdout)`：日志进 `.log`。
- 2026-09-25（PR #10 `5a5dc24`）参数误差日志标签 `[errors]` → `[param-err]`（避免误读为报错）。
- 2026-09-25（PR #13 `7765e0d`）输出/日志约定：内部诊断（pLBFGS/polish）由 `print` 改 `log`，
  结果表与可复现 tag 保留 `print`。

### 随机初值 / 输出文件
- 2026-09-25（PR #12 `279e5e8`）随机初值 `--seed <int>` / `FIT_SEED`（base 默认当前 Unix
  时间戳，run i 用 `base+i`；`--seed 42` 复现旧行为；启动打印可复现命令；`checkpoint.pt` 存 seed，
  `--resume` 以 checkpoint 为准）；best 点输出 `param_covariance.csv` / `param_correlation.csv`
  （完整 N×N，仅正常拟合、仅 best；log 只提示）。

### 代码整理
- 2026-09-25（PR #13）常量上移、删死代码/历史叙事、`_project_params_`→`_project_params`、
  去重 `generate_initial_params`（bit-exact）、补类型注解 + `ErrDict/RunResult`、
  `cfg` dict → `FitConfig(Mapping)` 兼容式 dataclass、`main()` 抽纯函数
  `_determine_base_seed/_seed_message`；**修 `--warm-start auto` 按 `--output-dir` 找
  `best_params.pt`**（旧版恒指 `results/`）。
- 2026-09-25（PR #14）polish 终态 verbose 在“全活跃”时的 `lmax` NameError；非 PD 且
  `err_mode=strict` 时的警告措辞；参数矩阵输出复用 best 误差（去掉重复 `[param-err]` 与重复 eigh）。

### 文档引用清理
- 2026-09-25（PR #11 `4b6b3a9`）去掉注释中指向仓库外/不存在的文档引用，以及内部编号标注。

### 路径口径（PR #21）
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
    改为「相对 config 目录」（与 `phsp_truth` 一致）。**cwd == config 目录时（生产作业的用法）
    全部是 no-op。**
  - 新增 `_ORIG_CWD`（记录 chdir 前的 cwd）；`_config_path_from_argv` 的 docstring 同步更正
    （原文那句"只影响 chdir"已过期）。
  - 本地单测 `example/tests/test_paths.py`（8 项，CPU）；GPU smoke 见 `../test16/`。
  - ⚠ 与上游分歧：上游 1fda180 仍是「main() 里 chdir」+ Data 段路径原样存字符串（`src/Config.cu`
    `parseData` 不做 `parent_path()`），因此上游没有"config 内相对路径统一"这一口径；将来合并
    上游时注意这一点。

---

## `example/plot.py`

- 2026-09-22（PR #2 `af0e72b`）应用 ruff-format。
- 2026-09-24（PR #7 `b1924f4`）`writeWeightFile` 的复数 dtype 按 `.so` 编译精度选择
  （`complex128`/`complex64`），修 “vector dtype must match .so complex precision”。
- 2026-09-29（PR #15 `1f66ceb`）**Dalitz(2D) pull/Fit 加入本底**：`total_fit_values = hfit+hbkg`；
  Pull 用 `(data-total)/√total`；Fit 面板画 total、标题有本底时 `Fit+Bkg`；`vmax=max(data,total)`。
  与 1D（`plot_combined_histogram_with_pull`）口径一致。
- 2026-09-29（PR #17，https://github.com/wwdws1/ctpwa/pull/17 ）修 `--params N` 路径：
  原先调用不存在的 `analysis.writeWeightFile`，且参数文件名/解析都不对 → 改为从
  `results/parameters.txt` 按 `# RUN: run_N` 分块正确解析（耦合 `±`/`(fixed)` 与共振态行），
  拼 `[Re, Im, θ]`（float64）后调 `analysis.writeResult`；失败**明确报错**、不再静默回退。

---

## `example/README.md`

- 2026-09-22（PR #1/#2/#3）随 reparam / err-mode / 默认优化器更新。
- 2026-09-23（PR #5 `43ac312`）修过期内容：默认优化器措辞、`--config` 已生效、环境变量补全、
  摘要列/系综、§3.5 来源标注。
- 2026-09-24（PR #6 `e4e0ff3`）FF/效率不再 PD 门控 + 大 `phsp_truth` 崩溃提示 + `--ff-only`。
- 2026-09-25（PR #9 `4d24aa6`）`--cal-ff`/`--cal-eff` 参数表与说明。
- 2026-09-25（PR #12 `279e5e8`）`--seed`/`FIT_SEED`、输出文件表加参数矩阵、seed 说明。
- 2026-09-25（PR #13 `2d9c761`）`--warm-start auto` 目录修正 + 输出/日志约定。
- 2026-09-29（PR #16）末尾新增 **§8 `plot.py` 使用说明**。
- 2026-09-29（PR #17）开头注明 `plot.py` 见 §8、变更历史见 `CHANGELOG.md`。
- 2026-10-22（PR #18）新增 **§9 参数削减 / 误差截断 / 边界诊断 / `--fit-mode`** 参数表与
  使用建议（含 `--reduce-params` 会替换最终解、生产默认不开的说明）。
- 2026-10-29（PR #19）新增 **§10 后端选择（CPU/GPU）与 `CTPWA_DEVICE`**；§1/§5 补 ctpwa
  版本要求与新门面环境变量；§7 备注补“正定判据变为可行子空间(KKT)”与“仓库根 `ctpwa/`
  会遮蔽已装 wheel”两条坑；§9 说明误差口径已改为可行子空间。
- 2026-10-29（PR #20）新增 **§11 MC 生成（`gen_phsp_mc.py` / `gen_toy_mc.py`）**。

---

## `example/gen_phsp_mc.py`（新增，PR #20）

- 2026-10-29（PR #20）**DIYMC P1 平坦相空间生成**（自包含单文件，不 import 本仓库其它文件）：
  `--config -n -o [--seed --format dat|root|npy --mother --mother-mass --weighted --chunk
  --force --no-check --check-events]`；`dat` 输出 = 每事件 n_final 行 `E px py pz`（12 位小数），
  末态顺序 = `Data.order`；内置自检（末态顺序 vs `Data.order`、守恒、质量壳、行数 = n×n_final、
  `weightMax`）；`--chunk` 分块追加写以支持千万级样本。**不需要 GPU，也不读数据文件**。
  文档里写清两条语义：平坦 PS ≠ 生产 `phsp_truth`；closure 用途需百万级样本（窄共振）。

---

## `example/gen_toy_mc.py`（新增，PR #20）

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

---

## `example/README.md`（续）

---

## `example/.pre-commit-config.yaml`

- 2026-09-22（PR #2 `fdbf26f`/`af0e72b`）新增：仅作用于 `example/`；含 ruff-format 等会改文件的
  hook；`ruff` 仅 `--select E9,F63,F7,F82`。
- 2026-09-25（PR #11 `4b6b3a9`）清理注释中指向仓库外文档的引用。

---

## 上游同步

- 2026-09-29（PR #16）并入上游 `BHXiang/ctpwa`（`a3946d5` → `68e0d0c`），**不涉及 `example/`**：
  - `68e0d0c` 修大 `phsp_truth` 下 `getFitFractions`/`getEfficiency` 崩溃与内存泄漏（issue #3）；
  - `7c59d36` trans 支持 N 个链名 + 每名字一个比值；ONE 占位质量不进振幅；
  - `754cd98` 发布 v0.3.9（`setup.py`）。
  - 涉及文件：`include/*`、`src/*`、`tests/configs/trans_multi*.yml`、`tests/test_trans_host.py`、`setup.py`。
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
    （含全部 v0.3.13 C++）。`50259f2` 只在生产配置存在“重复 `[J,P]` 块”时才影响结果，
    经查 `jlsp/pwa/all/single` 无此结构 → 现有结果不受影响。
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
  - `f82315a` `phsp_weights` 进入 NLL 归一化/梯度/Hessian（我们的生产 config 未用
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
    已在 `test14/sub.gpu` 与生产 `sub.gpu` 中显式设置（避免静默退 CPU）。
