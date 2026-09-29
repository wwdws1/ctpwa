# CHANGELOG（`example/`）

本文件记录对上游 `BHXiang/ctpwa` 的改动中、**`example/` 目录内**各文件的内容变化
（PR #1–#16）。其它目录随上游维护，不在本表；上游同步见文末。
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

---

## `example/plot.py`

- 2026-09-22（PR #2 `af0e72b`）应用 ruff-format。
- 2026-09-24（PR #7 `b1924f4`）`writeWeightFile` 的复数 dtype 按 `.so` 编译精度选择
  （`complex128`/`complex64`），修 “vector dtype must match .so complex precision”。
- 2026-09-29（PR #15 `1f66ceb`）**Dalitz(2D) pull/Fit 加入本底**：`total_fit_values = hfit+hbkg`；
  Pull 用 `(data-total)/√total`；Fit 面板画 total、标题有本底时 `Fit+Bkg`；`vmax=max(data,total)`。
  与 1D（`plot_combined_histogram_with_pull`）口径一致。

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
