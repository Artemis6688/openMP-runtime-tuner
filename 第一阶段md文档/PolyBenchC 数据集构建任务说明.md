# PolyBench/C 数据集构建任务说明文档

**项目：CPU-only 场景下 OpenMP runtime 参数自动调优**
**阶段：Intel Xeon 单平台第一阶段 / PolyBench 数据集构建 v0**

## 0. 文档目的

本文档用于说明当前要执行的**数据集构建任务**。

项目总体目标是：在 CPU-only 场景下，针对 Intel Xeon 平台，构建一个可扩展到异构 CPU 平台的 OpenMP runtime 参数自动调优环境。模型不允许自由改写程序源码，而是在受控的 OpenMP runtime action space 内推荐运行时参数配置；harness 负责 action validation、参数 materialization、程序运行、正确性检查、性能测量和 reward 计算。

当前本文档只聚焦第一批数据源：

```text
PolyBench/C
```

我们希望把 PolyBench/C 中的 benchmark kernel 改造成适合 OpenMP runtime 参数调优的数据集，然后在目标 Intel Xeon 机器上真实运行不同 OpenMP runtime 配置，收集：

```text
program + input + platform + OpenMP runtime action → correctness + runtime + reward
```

最终产物不是普通代码数据集，而是一个 **OpenMP runtime action-performance trial dataset**。

------

## 1. 当前任务的核心目标

当前任务不是训练模型，而是先构建数据底座。

目标是：

```text
以 PolyBench/C 为程序来源，
对每个 kernel 构建一个受控 OpenMP 版本，
在目标 Intel Xeon 机器上运行大量 OpenMP runtime 参数组合，
记录每个组合的正确性、运行时间、speedup、reward，
形成 raw trial dataset。
```

后续 SFT / DPO / RL 数据都从这个 raw trial dataset 派生。

------

## 2. PolyBench/C 的角色

PolyBench/C 是一个数值计算 benchmark suite，包含 30 个静态控制流 numerical computation kernels，覆盖 linear algebra、image processing、physics simulation、dynamic programming、statistics 等领域；它还提供统一的 kernel instrumentation、cache flushing、live-out array dump、parametric loop bounds、kernel marking 等机制，因此适合作为性能实验底座。([SourceForge](https://sourceforge.net/projects/polybench/files/README/download?utm_source=chatgpt.com))

但需要特别注意：

```text
PolyBench/C 原始版本主要是 sequential C kernel，
不是天然 OpenMP runtime tuning benchmark。
```

因此我们不能直接把原始 PolyBench/C 当作训练集。我们需要先做一层工程处理：

```text
PolyBench/C 原始 kernel
    ↓
只在 kernel_* 函数内部添加 OpenMP pragma
    ↓
编译 OpenMP 版本
    ↓
建立 correctness reference
    ↓
选择合适 input size
    ↓
在目标机器上运行不同 OpenMP runtime action
    ↓
记录 runtime / correctness / reward
    ↓
形成 raw trial dataset
```

------

## 3. 当前阶段的关键决策

### 3.1 当前先不训练模型

当前阶段只做：

```text
数据集构建
harness 准备
trial collection
数据质量检查
```

暂不直接进入：

```text
SFT
DPO
GRPO / PPO
```

原因是：SFT 和 DPO 必须建立在干净、可复现、经过质量过滤的 raw trial 数据之上。

------

### 3.2 当前先不划分 train / val / test

当前阶段先统一处理 PolyBench/C 的 30 个 kernel。

也就是说，当前 raw dataset 里可以先写：

```json
{
  "split": "unassigned"
}
```

等所有 kernel 的状态清楚之后，再冻结：

```text
train / val / test
```

这么做的原因是：现在还不知道哪些 kernel 能安全 OpenMP 化，哪些 kernel 正确性稳定，哪些 kernel runtime 太短或太长。过早划分 train / val / test 可能导致后面需要频繁调整。

但必须注意：一旦进入 SFT / DPO / evaluation 数据导出阶段，就必须按 `program_id` 或 `program_family` 划分，不能按 trial row 随机划分。

错误做法：

```text
同一个 kernel 的不同 action trial 一部分进入 train，一部分进入 test。
```

正确做法：

```text
一个 kernel 的所有 trial 必须全部属于 train / val / test 中的一个 split。
```

------

### 3.3 当前先使用绝对 OpenMP runtime action，不强制归一化

项目长期目标可以使用 normalized action，例如：

```text
0.5C
1.0C
1NUMA
1Socket
```

但当前阶段是在一台具体 Intel Xeon 机器上采集第一批实验数据，因此先使用绝对 action 更直接。

当前 trial 里先记录：

```json
{
  "OMP_NUM_THREADS": 64,
  "OMP_PROC_BIND": "close",
  "OMP_PLACES": "cores",
  "OMP_SCHEDULE": "static",
  "OMP_DYNAMIC": "FALSE",
  "OMP_WAIT_POLICY": "PASSIVE",
  "OMP_MAX_ACTIVE_LEVELS": 1
}
```

同时建议额外记录可后处理字段：

```json
{
  "derived_thread_features": {
    "threads_ratio_physical_cores": 1.0,
    "threads_ratio_logical_cpus": 0.5,
    "threads_per_socket_ratio": 2.0,
    "threads_per_numa_ratio": 2.0
  }
}
```

这样当前实验不受 normalized action 限制，未来仍然可以分析是否有必要把动作空间转成归一化形式。

------

### 3.4 当前尝试处理全部 30 个 PolyBench/C kernel

目标是：

```text
30 个 kernel 都尝试处理。
```

但这不意味着必须强行并行化所有 kernel。

对于每个 kernel，结果可以是：

```text
ready
partial
excluded_v0
failed_compile
failed_correctness
too_short
too_slow
too_noisy
```

解释：

```text
ready:
    可以安全 OpenMP 化，正确性通过，runtime 稳定，可以进入正式 trial。

partial:
    只安全并行化了一部分 loop，但仍可运行、可测量，可以视情况进入 trial。

excluded_v0:
    当前阶段不适合作为 v0 数据集，例如依赖太复杂，无法安全添加 OpenMP pragma。

failed_compile:
    编译失败。

failed_correctness:
    OpenMP 后输出不正确。

too_short:
    runtime 太短，测量噪声过大。

too_slow:
    runtime 太长，采样成本太高。

too_noisy:
    多次运行波动过大。
```

------

## 4. 当前必须先采集目标机器信息

在开始 PolyBench 处理之前，必须先采集目标机器 profile。原因是 OpenMP runtime 参数空间依赖于：

```text
物理核心数
逻辑 CPU 数
socket 数
NUMA node 数
当前 job 可用 CPU 集合
compiler
OpenMP runtime implementation
frequency / turbo / governor 状态
```

项目第一阶段虽然只聚焦 Intel Xeon，但仍然要求使用结构化 platform profile，这也是后续扩展到 AMD、ARM/鲲鹏、SW 等平台的基础。

### 4.1 建议采集脚本

创建文件：

```bash
collect_platform_profile.sh
```

内容如下：

```bash
#!/usr/bin/env bash
set -euo pipefail

OUT=${1:-platform_profile_raw}
mkdir -p "$OUT"

uname -a > "$OUT/uname.txt"

lscpu > "$OUT/lscpu.txt"
lscpu --json > "$OUT/lscpu.json"
lscpu -e=CPU,CORE,SOCKET,NODE,ONLINE,MAXMHZ,MINMHZ,MHZ > "$OUT/lscpu_extended.txt"

nproc --all > "$OUT/nproc_all.txt"
nproc > "$OUT/nproc_available.txt"

cat /proc/cpuinfo > "$OUT/cpuinfo.txt"

numactl --hardware > "$OUT/numactl_hardware.txt" 2>&1 || true
numactl --show > "$OUT/numactl_show.txt" 2>&1 || true

taskset -pc $$ > "$OUT/current_taskset.txt"
grep Cpus_allowed_list /proc/self/status > "$OUT/cpus_allowed_list.txt"

gcc --version > "$OUT/gcc_version.txt" 2>&1 || true
g++ --version > "$OUT/gxx_version.txt" 2>&1 || true
clang --version > "$OUT/clang_version.txt" 2>&1 || true
icx --version > "$OUT/icx_version.txt" 2>&1 || true
icc --version > "$OUT/icc_version.txt" 2>&1 || true
ldd --version > "$OUT/ldd_version.txt" 2>&1 || true

env | sort > "$OUT/env_all.txt"
env | sort | grep -E '^(OMP|GOMP|KMP|MKL|OPENBLAS|SLURM|PBS|LSB|LD_LIBRARY_PATH)=' > "$OUT/env_relevant.txt" || true

cat > "$OUT/omp_probe.c" <<'EOF'
#include <stdio.h>
#include <omp.h>

int main() {
#ifdef _OPENMP
  printf("_OPENMP=%d\n", _OPENMP);
#else
  printf("_OPENMP=not_defined\n");
#endif

  printf("omp_get_num_procs=%d\n", omp_get_num_procs());
  printf("omp_get_max_threads=%d\n", omp_get_max_threads());
  printf("omp_get_dynamic=%d\n", omp_get_dynamic());
  printf("omp_get_nested=%d\n", omp_get_nested());
  printf("omp_get_max_active_levels=%d\n", omp_get_max_active_levels());

#pragma omp parallel
  {
#pragma omp single
    {
      printf("actual_parallel_threads=%d\n", omp_get_num_threads());
    }
  }

  return 0;
}
EOF

gcc -O2 -fopenmp "$OUT/omp_probe.c" -o "$OUT/omp_probe_gcc"
ldd "$OUT/omp_probe_gcc" > "$OUT/omp_probe_gcc_ldd.txt"
OMP_DISPLAY_ENV=VERBOSE "$OUT/omp_probe_gcc" > "$OUT/omp_probe_gcc_stdout.txt" 2> "$OUT/omp_probe_gcc_stderr.txt"

cpupower frequency-info > "$OUT/cpupower_frequency_info.txt" 2>&1 || true
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor > "$OUT/scaling_governor_cpu0.txt" 2>&1 || true
cat /sys/devices/system/cpu/intel_pstate/no_turbo > "$OUT/intel_pstate_no_turbo.txt" 2>&1 || true

for f in /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq; do
  echo "$f $(cat $f 2>/dev/null)"
done > "$OUT/scaling_cur_freq_all.txt" || true

tar czf "${OUT}.tar.gz" "$OUT"
echo "Wrote ${OUT}.tar.gz"
```

运行：

```bash
bash collect_platform_profile.sh
```

完成后应得到：

```text
platform_profile_raw/
platform_profile_raw.tar.gz
```

### 4.2 需要从机器信息中提取的字段

后续应整理出一个 `platform_profile.json`，至少包括：

```json
{
  "platform_id": "intel_xeon_single_machine_v0",
  "platform_label": "Intel_Xeon",
  "vendor": "Intel",
  "isa": "x86_64",
  "cpu_model": "...",
  "num_sockets": 2,
  "numa_nodes": 2,
  "physical_cores": 64,
  "logical_cpus": 128,
  "threads_per_core": 2,
  "cores_per_socket": 32,
  "cores_per_numa_node": 32,
  "available_cpus": "...",
  "compiler": {
    "name": "gcc",
    "version": "..."
  },
  "openmp_runtime": {
    "name": "libgomp",
    "version": "..."
  },
  "os": "linux",
  "kernel": "...",
  "frequency_policy": "fixed_or_unknown",
  "turbo": "enabled_or_disabled_or_unknown"
}
```

------

## 5. PolyBench/C 源码处理原则

### 5.1 总原则

我们只允许把 PolyBench/C 改造成 OpenMP runtime tuning workload。

不允许把这一步变成算法优化、编译优化或源码重写任务。

也就是说：

```text
允许：
    添加 OpenMP pragma

不允许：
    改算法
    改循环边界
    改数组下标
    改数据结构
    做 blocking / tiling / interchange / fusion / fission
    引入 task
    引入 nested parallelism
```

当前项目研究的是：

```text
在固定程序版本上调 OpenMP runtime 参数。
```

不是研究：

```text
如何自动改写程序源码。
```

这与项目第一阶段设定一致：模型不允许自由改写程序源码，只在受控 OpenMP runtime action space 中输出参数配置。

------

## 6. 给大模型进行 PolyBench OpenMP 化的严格规则

下面这一段可以直接复制给负责修改 PolyBench kernel 的 AI。

------

### 6.1 OpenMP annotation 任务说明

你是一个 OpenMP C 程序并行化助手。你的任务是只在 PolyBench/C 的 `kernel_*` 函数内部添加 OpenMP pragma，使程序可用于 OpenMP runtime 参数调优实验。

你的目标不是最大化手写并行性能，而是构造一个**语义正确、改动受控、适合 runtime 参数调优**的 OpenMP 版本。

------

### 6.2 硬性限制

1. **只允许修改 `kernel_\*` 函数内部。**

   禁止修改：

   ```text
   main
   init_array
   print_array
   PolyBench instrumentation
   宏定义
   数据分配
   计时代码
   输出代码
   ```

2. **不允许改变算法语义。**

   禁止修改：

   ```text
   循环边界
   数组下标
   计算表达式
   输出数组含义
   初始化逻辑
   ```

3. **不允许复杂 source transformation。**

   禁止：

   ```text
   loop tiling
   loop blocking
   loop interchange
   loop fusion
   loop fission
   数据布局变换
   引入额外大数组
   算法替换
   wavefront transformation
   prefix-sum transformation
   ```

4. **只允许添加以下 OpenMP 构造：**

   ```c
   #pragma omp parallel for schedule(runtime)
   ```

   以及必要 clause：

   ```text
   private(...)
   firstprivate(...)
   reduction(...)
   ```

   默认不使用 `collapse`。除非明确要求并能证明安全，否则不要添加 `collapse(n)`。

5. **所有被并行化的 loop 必须使用 `schedule(runtime)`。**

   这是为了让 harness 设置的：

   ```text
   OMP_SCHEDULE
   ```

   能真实影响程序运行。

6. **不允许 nested parallelism。**

   禁止：

   ```text
   在 parallel region 内创建新的 parallel region
   omp task
   omp sections
   omp target
   omp teams
   ```

7. **默认不使用 `omp simd`。**

   当前任务调优 OpenMP runtime，不研究 SIMD directive。

8. **只并行化可以证明没有 loop-carried dependence 的循环。**

   如果循环存在以下情况，则不要并行化：

   ```text
   前后迭代读写依赖
   recurrence
   triangular solve dependence
   dynamic programming dependence
   stencil in-place dependence
   迭代间写同一数组元素
   ```

9. **reduction 必须谨慎使用。**

   只有当变量明确是 scalar accumulation 时，才允许使用：

   ```c
   reduction(+:sum)
   ```

   对浮点 reduction 必须说明：并行规约可能导致微小数值差异，correctness check 需要 tolerance。

10. **不能为了增加并行区域而添加不安全 pragma。**

    如果不能证明安全，就保留串行。

------

### 6.3 输出要求

对每个 kernel，AI 必须输出三部分。

#### A. 修改后的完整 `kernel_*` 函数

只输出被修改后的 `kernel_*` 函数，不要重写整个文件，除非被明确要求。

#### B. 每个 pragma 的安全性说明

对每个添加的 OpenMP pragma，说明：

```text
并行化的是哪一个 loop
loop index 是什么
private 变量有哪些
是否使用 reduction
为什么没有 loop-carried dependence
是否使 OMP_SCHEDULE 生效
```

#### C. JSON summary

格式如下：

```json
{
  "kernel_name": "gemm",
  "openmp_status": "ready",
  "modified_scope": "kernel_gemm_only",
  "num_omp_parallel_for": 1,
  "schedule_runtime_enabled": true,
  "has_nested_parallel": false,
  "has_task": false,
  "has_reduction": false,
  "has_collapse": false,
  "requires_correctness_tolerance": true,
  "parallelized_loops": [
    {
      "loop_indices": ["i"],
      "pragma": "#pragma omp parallel for schedule(runtime) private(j,k)",
      "reason": "Each i iteration writes a disjoint row of the output matrix and only reads shared input arrays."
    }
  ],
  "skipped_loops": [
    {
      "loop_indices": ["k"],
      "reason": "Inner loop contributes to the same output element and is not separately parallelized to avoid races."
    }
  ],
  "notes": []
}
```

------

## 7. 源码轻微改写的策略

当前 v0 阶段建议：

```text
允许 OpenMP annotation
不允许算法级 source rewrite
```

### 7.1 允许的修改

```text
添加 #pragma omp parallel for schedule(runtime)
添加 private / firstprivate / reduction
添加少量注释
必要时添加局部 scalar private 说明
```

### 7.2 不建议当前阶段做的修改

```text
拆 loop
合并 loop
换 loop 顺序
引入临时大数组
改变算法结构
wavefront 改写
prefix-sum 改写
blocking / tiling
```

### 7.3 原因

接受复杂源码改写的优点是：

```text
可以让更多 kernel 变得并行
可能获得更高性能
```

但风险是：

```text
性能变化可能来自源码重写，而不是 runtime 参数
不同 kernel 改写程度不一致，数据分布混乱
后续模型难以判断是程序特征还是改写方式导致性能变化
正确性验证成本大幅增加
```

所以 v0 阶段建议保持严格边界：

```text
只做 OpenMP pragma annotation。
```

如果未来需要源码变体，可以单独建立：

```text
manual_omp_v0
collapse_v1
rewrite_v1
wavefront_v1
```

但它们必须作为不同 program variant 记录，并且同一 `program_family` 后续划分 train/test 时不能泄漏。

------

## 8. Input size 策略

PolyBench/C 的问题规模通常通过编译期宏控制，例如：

```text
MINI_DATASET
SMALL_DATASET
MEDIUM_DATASET
LARGE_DATASET
EXTRALARGE_DATASET
```

当前策略是：

```text
默认从 LARGE_DATASET 开始，
根据 baseline runtime 自适应调整。
```

### 8.1 为什么不直接统一 LARGE_DATASET

统一 `LARGE_DATASET` 的优点：

```text
简单
一致
容易复现
```

但问题是：

```text
不同 kernel 的 LARGE_DATASET runtime 可能差异很大。
有的 kernel 太短，测量噪声占比高。
有的 kernel 太长，采集 action trial 成本过高。
```

OpenMP runtime tuning 对噪声敏感，因此每个 kernel 应该选择一个合适 input size，使 baseline runtime 落入目标窗口。

------

### 8.2 建议 runtime 目标窗口

v0 阶段建议目标：

```text
baseline median runtime: 0.5s - 30s
```

宽松接受范围：

```text
0.2s - 60s
```

### 8.3 自适应选择规则

对每个 kernel：

```text
1. 先用 LARGE_DATASET 编译并运行 canonical baseline。

2. 如果 median runtime < 0.2s：
       尝试 EXTRALARGE_DATASET。

3. 如果 median runtime 在 0.5s - 30s：
       接受该 size。

4. 如果 median runtime 在 0.2s - 0.5s：
       可以接受，但标记为 short_runtime。

5. 如果 median runtime > 60s：
       降到 MEDIUM_DATASET。

6. 如果 MEDIUM_DATASET 仍然 > 60s：
       降到 SMALL_DATASET 或标记 excluded_v0。

7. 如果最大 size 仍然太短：
       使用最大 size，但标记 noisy_candidate。

8. 如果所有 size 都太慢：
       暂时不进入 v0 trial。
```

------

## 9. OpenMP runtime action space：absolute v0

当前阶段先使用 absolute action。

最终 action 直接对应环境变量：

```text
OMP_NUM_THREADS
OMP_PROC_BIND
OMP_PLACES
OMP_SCHEDULE
OMP_DYNAMIC
OMP_WAIT_POLICY
OMP_MAX_ACTIVE_LEVELS
```

### 9.1 `OMP_NUM_THREADS`

具体候选值需要等机器 profile 采集后确定。

假设机器为：

```text
64 physical cores
128 logical CPUs
2 sockets
2 NUMA nodes
```

可以初步使用：

```text
1
2
4
8
16
32
48
64
96
128
```

但如果当前 job 只允许使用部分 CPU，则不能超过当前可用 CPU 集合。

通用生成规则：

```text
1
2
4
8
16
physical_cores / 4
physical_cores / 2
physical_cores
1.5 * physical_cores
logical_cpus
cores_per_socket
cores_per_numa_node
```

然后去重、排序、过滤非法值。

------

### 9.2 其他 OpenMP 变量候选

初始 v0 可用：

```json
{
  "OMP_PROC_BIND": [
    "false",
    "close",
    "spread"
  ],
  "OMP_PLACES": [
    "threads",
    "cores",
    "sockets"
  ],
  "OMP_SCHEDULE": [
    "static",
    "dynamic,1",
    "dynamic,16",
    "dynamic,64",
    "guided,16",
    "guided,64",
    "auto"
  ],
  "OMP_DYNAMIC": [
    "FALSE",
    "TRUE"
  ],
  "OMP_WAIT_POLICY": [
    "PASSIVE",
    "ACTIVE"
  ],
  "OMP_MAX_ACTIVE_LEVELS": [
    1
  ]
}
```

当前 PolyBench v0 不引入 nested parallelism，因此：

```text
OMP_MAX_ACTIVE_LEVELS 固定为 1。
```

------

## 10. Baseline 定义

每个 `program + input + platform` 都必须有 canonical baseline。

建议 baseline：

```bash
export OMP_NUM_THREADS=<physical_cores_or_available_cores>
export OMP_PROC_BIND=close
export OMP_PLACES=cores
export OMP_SCHEDULE=static
export OMP_DYNAMIC=FALSE
export OMP_WAIT_POLICY=PASSIVE
export OMP_MAX_ACTIVE_LEVELS=1
```

如果当前进程受 cpuset / Slurm 限制，则 `<physical_cores_or_available_cores>` 应该使用当前 job 可用 CPU 数，而不是整机物理核心数。

------

## 11. Reward 定义

每条 trial 有：

```text
T_base   = canonical baseline median runtime
T_action = 当前 action median runtime
```

reward：

```text
if invalid_action:
    reward = -1.0
elif correctness_fail:
    reward = -1.0
elif timeout:
    reward = -0.5 或 -1.0
else:
    reward = log(T_base / T_action)
```

含义：

```text
reward > 0: 比 baseline 快
reward ≈ 0: 和 baseline 接近
reward < 0: 比 baseline 慢
```

这与项目第一阶段文档中的 correctness-gated log speedup 设计一致。

------

## 12. Trial collection 策略

当前不建议 full grid search，因为组合数会很大。

建议分两轮。

### 12.1 Round 0：sanity trial

目标：

```text
检查编译、正确性、runtime、噪声、timeout、action materialization 是否正常。
```

规模：

```text
每个 kernel 约 20 个 action
每个 action 1 次 warmup + 3 次 measured runs
```

Round 0 后筛掉：

```text
failed_compile
failed_correctness
too_short
too_slow
too_noisy
```

### 12.2 Round 1：正式 trial

目标：

```text
构建 v0 raw trial dataset。
```

规模建议：

```text
每个 ready / partial kernel 120-180 个 action
每个 action 1 次 warmup + 5 次 measured runs
```

如果 30 个 kernel 全部进入正式 trial，且每个 kernel 150 个 action：

```text
raw trial rows = 30 × 150 = 4500
measured executions = 4500 × 5 = 22500
```

这比之前的 `30 × 72 = 2160` 更适合后续 SFT / DPO 派生数据。

------

## 13. 2160 条 trial 是否能进入 SFT

如果按：

```text
30 kernels × 72 actions = 2160 raw trials
```

它可以用于：

```text
v0 pipeline 验证
SFT sanity check
DPO 数据格式验证
模型合法输出验证
```

但不能直接等价于“正式 SFT 数据集已经准备好”。

原因是 raw trials 还需要经过：

```text
质量过滤
baseline 计算
speedup / reward 计算
per-kernel ranking
top-K action selection
history-state 派生
SFT prompt 构造
DPO chosen/rejected pair 构造
train/val/test split 冻结
```

### 13.1 Raw trial 到 SFT 的处理流程

```text
raw trials
    ↓
filter invalid / incorrect / timeout / noisy trials
    ↓
group by program_id + input_id + platform_id
    ↓
compute baseline runtime
    ↓
compute speedup and reward
    ↓
rank actions by reward
    ↓
select top-K actions
    ↓
construct zero-history SFT samples
    ↓
construct trial-history SFT samples
    ↓
export sft.jsonl
```

### 13.2 为什么不能直接把每条 trial 变成 SFT target

因为 SFT 应该学习推荐好 action，而 raw trials 里包含：

```text
好 action
一般 action
慢 action
错误 action
timeout action
噪声大的 action
```

不能无差别用于模仿学习。

------

## 14. Trial 数据格式

核心文件建议包括：

```text
platform_profile.json
polybench_programs_raw.jsonl
polybench_actions_absolute_v0.jsonl
polybench_trials_raw.jsonl
```

------

### 14.1 `platform_profile.json`

示例：

```json
{
  "platform_id": "intel_xeon_single_machine_v0",
  "platform_label": "Intel_Xeon",
  "vendor": "Intel",
  "isa": "x86_64",
  "cpu_model": "...",
  "num_sockets": 2,
  "numa_nodes": 2,
  "physical_cores": 64,
  "logical_cpus": 128,
  "threads_per_core": 2,
  "cores_per_socket": 32,
  "cores_per_numa_node": 32,
  "available_cpus": "0-127",
  "compiler": {
    "name": "gcc",
    "version": "..."
  },
  "openmp_runtime": {
    "name": "libgomp",
    "version": "..."
  },
  "os": "linux",
  "kernel": "...",
  "frequency_policy": "unknown",
  "turbo": "unknown"
}
```

------

### 14.2 `polybench_programs_raw.jsonl`

一行一个 kernel。

```json
{
  "program_id": "polybench_gemm_omp_v0",
  "program_family": "polybench_gemm",
  "kernel_name": "gemm",
  "source_dataset": "PolyBench/C",
  "source_version": "4.2.1",
  "category": "linear-algebra/blas",
  "language": "C",
  "source_file": "benchmarks/polybench_omp_v0/linear-algebra/blas/gemm/gemm.c",
  "openmp_variant": "manual_omp_annotation_v0",
  "openmp_status": "ready",
  "num_omp_parallel_for": 1,
  "schedule_runtime_enabled": true,
  "has_nested_parallel": false,
  "has_task": false,
  "has_reduction": false,
  "has_collapse": false,
  "selected_dataset_macro": "LARGE_DATASET",
  "input_size_status": "accepted",
  "baseline_runtime_median_sec": 1.83,
  "split": "unassigned"
}
```

------

### 14.3 `polybench_actions_absolute_v0.jsonl`

一行一个 action。

```json
{
  "action_id": "action_000001",
  "action_space_version": "absolute_openmp_runtime_v0",
  "env": {
    "OMP_NUM_THREADS": "64",
    "OMP_PROC_BIND": "close",
    "OMP_PLACES": "cores",
    "OMP_SCHEDULE": "static",
    "OMP_DYNAMIC": "FALSE",
    "OMP_WAIT_POLICY": "PASSIVE",
    "OMP_MAX_ACTIVE_LEVELS": "1"
  }
}
```

------

### 14.4 `polybench_trials_raw.jsonl`

一行一个 `program + input + platform + action` 的聚合结果。

```json
{
  "trial_id": "polybench_trial_000001",
  "dataset_version": "polybench_omp_runtime_v0",
  "timestamp": "2026-04-29T12:00:00-07:00",
  "program_id": "polybench_gemm_omp_v0",
  "program_family": "polybench_gemm",
  "kernel_name": "gemm",
  "split": "unassigned",
  "input_id": "large",
  "dataset_macro": "LARGE_DATASET",
  "platform_id": "intel_xeon_single_machine_v0",
  "action_id": "action_000001",
  "absolute_action": {
    "OMP_NUM_THREADS": "64",
    "OMP_PROC_BIND": "close",
    "OMP_PLACES": "cores",
    "OMP_SCHEDULE": "static",
    "OMP_DYNAMIC": "FALSE",
    "OMP_WAIT_POLICY": "PASSIVE",
    "OMP_MAX_ACTIVE_LEVELS": "1"
  },
  "derived_thread_features": {
    "threads_ratio_physical_cores": 1.0,
    "threads_ratio_logical_cpus": 0.5,
    "threads_per_socket_ratio": 2.0,
    "threads_per_numa_ratio": 2.0
  },
  "validation": {
    "valid_action": true,
    "validation_notes": []
  },
  "execution": {
    "compile_success": true,
    "correct": true,
    "timeout": false,
    "exit_code": 0
  },
  "measurement": {
    "warmup_runs": 1,
    "runtime_runs_sec": [
      1.83,
      1.81,
      1.84,
      1.82,
      1.83
    ],
    "runtime_median_sec": 1.83,
    "runtime_mean_sec": 1.826,
    "runtime_std_sec": 0.0114,
    "runtime_cv": 0.0062
  },
  "baseline": {
    "baseline_action_id": "canonical_openmp_baseline_v0",
    "baseline_runtime_median_sec": 2.10
  },
  "objective": {
    "speedup": 1.1475,
    "reward": 0.1376,
    "reward_type": "correctness_gated_log_speedup"
  }
}
```

------

## 15. Correctness 策略

PolyBench/C 支持 live-out array dump，可用于构建 reference output。([SourceForge](https://sourceforge.net/projects/polybench/files/README/download?utm_source=chatgpt.com))

建议流程：

```text
1. 对原始 sequential PolyBench kernel 编译 reference binary。
2. 使用同一个 dataset macro 运行并 dump output。
3. 保存 reference output。
4. 对 OpenMP 版本在每个 action 下运行 dump 或 checksum。
5. 与 reference output 比较。
```

浮点程序不建议做严格 byte-level diff，因为 OpenMP reduction 或调度顺序可能造成微小数值差异。

建议：

```text
浮点输出：
    absolute tolerance + relative tolerance

整数输出：
    exact match
```

初始 tolerance 可设：

```text
abs_tol = 1e-6
rel_tol = 1e-5
```

之后按 kernel 调整。

------

## 16. 数据质量过滤规则

正式进入 SFT / DPO 之前，至少过滤：

```text
compile_success == true
valid_action == true
correct == true
timeout == false
runtime_median_sec > 0
baseline_runtime_median_sec > 0
runtime_cv <= 0.05
```

对少数天然波动较大的 kernel，可放宽到：

```text
runtime_cv <= 0.10
```

但必须标记：

```json
{
  "quality_flag": "noisy_but_accepted"
}
```

不应让高噪声数据无标记进入训练集。

------

## 17. 后续 SFT 数据如何从 raw trials 派生

SFT 不直接使用所有 raw trials，而是使用每个 kernel 中表现较好的 action。

### 17.1 Zero-history SFT

输入：

```text
program features
platform profile
action schema
```

输出：

```text
top-K high reward actions
```

示例：

```json
{
  "sample_id": "sft_zero_history_000001",
  "messages": [
    {
      "role": "system",
      "content": "You are an OpenMP runtime tuning policy. Output only valid JSON."
    },
    {
      "role": "user",
      "content": {
        "task": "Recommend top-3 OpenMP runtime configurations for this program and platform.",
        "platform_profile": "...",
        "program_features": "...",
        "action_space_version": "absolute_openmp_runtime_v0"
      }
    },
    {
      "role": "assistant",
      "content": [
        {
          "OMP_NUM_THREADS": "64",
          "OMP_PROC_BIND": "close",
          "OMP_PLACES": "cores",
          "OMP_SCHEDULE": "static",
          "OMP_DYNAMIC": "FALSE",
          "OMP_WAIT_POLICY": "PASSIVE",
          "OMP_MAX_ACTIVE_LEVELS": "1"
        }
      ]
    }
  ]
}
```

### 17.2 Trial-history SFT

为了扩充样本量，可以构造带历史反馈的训练样本。

输入：

```text
program features
platform profile
action schema
trial history
remaining budget
current best reward
```

输出：

```text
next top-K candidate actions
```

这更贴近最终部署流程：模型提出候选，harness 小预算实测，逐步找到好配置。

------

## 18. 后续 DPO 数据如何从 raw trials 派生

DPO 样本从同一个：

```text
program_id + input_id + platform_id
```

下构造 chosen / rejected pair。

要求：

```text
chosen.correct == true
rejected 可以是 correct 但慢，也可以是 timeout / incorrect / invalid
chosen.reward - rejected.reward >= margin
```

建议初始 margin：

```text
0.05 或 0.10
```

示例：

```json
{
  "sample_id": "dpo_000001",
  "prompt": {
    "task": "Choose a high-performance OpenMP runtime configuration.",
    "platform_profile": "...",
    "program_features": "...",
    "action_space_version": "absolute_openmp_runtime_v0"
  },
  "chosen": {
    "OMP_NUM_THREADS": "64",
    "OMP_PROC_BIND": "close",
    "OMP_PLACES": "cores",
    "OMP_SCHEDULE": "static",
    "OMP_DYNAMIC": "FALSE",
    "OMP_WAIT_POLICY": "PASSIVE",
    "OMP_MAX_ACTIVE_LEVELS": "1"
  },
  "rejected": {
    "OMP_NUM_THREADS": "128",
    "OMP_PROC_BIND": "false",
    "OMP_PLACES": "threads",
    "OMP_SCHEDULE": "dynamic,1",
    "OMP_DYNAMIC": "TRUE",
    "OMP_WAIT_POLICY": "ACTIVE",
    "OMP_MAX_ACTIVE_LEVELS": "1"
  },
  "metadata": {
    "chosen_reward": 0.1376,
    "rejected_reward": -0.2231,
    "reward_margin": 0.3607
  }
}
```

------

## 19. 最终 train / val / test 划分策略

当前阶段先不划分，但在导出训练数据之前必须划分。

### 19.1 三者区别

```text
train:
    用于训练模型参数。

val:
    用于选择 checkpoint、调 prompt、调 DPO 参数、调数据过滤规则。
    不参与梯度更新，但会影响开发决策。

test:
    最终评估用。
    不应该用于调参、调 prompt、选 checkpoint。
```

### 19.2 划分单位

必须按：

```text
program_id
```

或更严格按：

```text
program_family
```

划分。

不能按 trial row 随机划分。

### 19.3 初始建议

如果最后 30 个 kernel 大多数 ready，可以用：

```text
train: 21 kernels
val:   4 kernels
test:  5 kernels
```

如果 ready kernel 少于 30 个，则需要根据实际 ready 数量重新分配。

------

## 20. 当前推荐执行顺序

当前完整执行顺序如下：

```text
Step 0:
    采集目标机器 platform profile。

Step 1:
    固定 PolyBench/C 版本。
    建议使用 PolyBench/C 4.2.1。

Step 2:
    复制原始源码到工作目录。
    不直接修改 third_party 原始源码。

Step 3:
    建立 polybench_programs_raw.jsonl。
    30 个 kernel 全部登记，split 先写 unassigned。

Step 4:
    对每个 kernel 调用 AI 进行 OpenMP annotation。
    必须遵守“只修改 kernel_* 函数内部”的规则。

Step 5:
    编译每个 OpenMP kernel。

Step 6:
    构建 reference output。
    检查 correctness。

Step 7:
    做 input size profiling。
    默认 LARGE_DATASET，自适应选择最终 dataset macro。

Step 8:
    确定 absolute OpenMP action space。
    其中 OMP_NUM_THREADS 根据机器 profile 生成。

Step 9:
    运行 Round 0 sanity trials。
    每个 kernel 约 20 actions。

Step 10:
    质量检查。
    标记 ready / partial / excluded_v0 / failed_correctness / too_noisy 等状态。

Step 11:
    对 ready / partial kernel 运行 Round 1 正式 trials。
    每个 kernel 120-180 actions。

Step 12:
    生成 polybench_trials_raw.jsonl 或 parquet。

Step 13:
    分析 reward 分布、runtime 分布、action 效果、噪声。

Step 14:
    冻结 train / val / test split。

Step 15:
    从 raw trials 派生 SFT / DPO / eval 数据。
```

------

## 21. 当前阶段的非目标

当前阶段不做：

```text
不训练最终模型
不做 DPO 正式实验
不做 GRPO / PPO
不引入 AMD / ARM / SW
不引入 KMP_* / GOMP_* runtime-specific 参数
不让模型自由改写程序源码
不做 compiler flag tuning
不做 OpenMP directive tuning
不做源码级优化搜索
```

当前阶段只做：

```text
PolyBench/C → controlled OpenMP workload → action trial dataset
```

------

## 22. 当前阶段完成标准

当以下条件满足时，可以认为 PolyBench/C 数据集构建 v0 完成：

```text
1. 已采集并整理 platform_profile.json。

2. 30 个 PolyBench/C kernel 已全部登记到 registry。

3. 每个 kernel 都有明确状态：
       ready / partial / excluded_v0 / failed_xxx。

4. ready / partial kernel 已完成 OpenMP annotation。

5. 每个 ready / partial kernel 已完成 correctness check。

6. 每个 ready / partial kernel 已确定 input size。

7. 已生成 absolute action space。

8. 已完成 Round 0 sanity trial。

9. 已完成 Round 1 formal trial。

10. 已生成 raw trial dataset。

11. 每条 trial 包含：
       program_id
       platform_id
       input_id
       action
       runtime runs
       runtime median
       correctness
       timeout
       baseline runtime
       speedup
       reward

12. 已完成数据质量过滤报告。

13. 已准备好进入 SFT / DPO 数据导出阶段。
```

------

## 23. 一句话总结

当前 PolyBench/C 数据集构建任务的本质是：

```text
把 PolyBench/C 的 30 个 sequential numerical kernels，
在严格限制源码改动的前提下改造成 OpenMP runtime tuning workloads，
然后在一台目标 Intel Xeon 机器上用大量绝对 OpenMP runtime 参数组合真实运行，
收集 correctness-gated runtime reward，
形成后续 SFT / DPO / RL 训练与评估所需的 raw trial dataset。
```

当前最先要做的具体动作是：

```text
先采集目标机器 platform profile。
```

然后再决定：

```text
OMP_NUM_THREADS 候选集合
baseline 定义
action sampling 范围
PolyBench input size
正式 trial collection 规模
```