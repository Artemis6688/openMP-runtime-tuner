------

# CPU-only 场景下的 OpenMP runtime 参数自动调优：Intel Xeon 单平台第一阶段版本

本研究面向 **CPU-only 场景下的 OpenMP runtime 参数自动调优**。当前第一阶段聚焦 **Intel Xeon 平台**，暂不直接覆盖 AMD、ARM/鲲鹏、SW 等其他 CPU 架构。系统目标是构建一个可扩展到异构 CPU 平台的 **RL-compatible execution environment**，并在 Intel Xeon 上验证基于小型 LLM policy 的 OpenMP runtime 参数自动推荐与优化流程。

本研究不允许模型自由改写程序源码，而是将调优问题限制在一个受控、可验证的 OpenMP runtime action space 内。模型需要根据给定程序的程序特征、目标平台的 profile 信息、合法 action schema，以及可选的历史 trial 反馈，直接输出一组 OpenMP runtime 参数配置。系统随后通过 harness 在目标机器上执行 action validation、参数 materialization、程序运行、正确性检查、性能测量和 reward 计算。

------

## 1. 研究目标

当前阶段的目标是：

```text
给定一个 OpenMP 程序和 Intel Xeon 平台信息，
训练一个小型 LLM policy，
使其能够直接推荐 OpenMP runtime 参数配置，
并通过少量真实 trial 在目标机器上找到较优运行配置。
```

也就是说，系统最终希望实现：

```text
输入：
    OpenMP 程序 / 程序摘要
    Intel Xeon platform profile
    OpenMP runtime action schema
    可选 trial history

输出：
    推荐的 OpenMP runtime 参数配置
```

示例输出：

```json
{
  "threads": "1.0C",
  "proc_bind": "close",
  "places": "cores",
  "schedule": "static",
  "dynamic": false,
  "wait_policy": "passive",
  "max_active_levels": 1
}
```

其中 `threads = 1.0C` 表示使用全部物理核心数，最终由 harness 根据平台 profile 转换成真实环境变量，例如：

```bash
export OMP_NUM_THREADS=64
export OMP_PROC_BIND=close
export OMP_PLACES=cores
export OMP_SCHEDULE=static
export OMP_DYNAMIC=FALSE
export OMP_WAIT_POLICY=PASSIVE
export OMP_MAX_ACTIVE_LEVELS=1
```

------

## 2. 当前阶段的研究范围

当前第一阶段只做：

```text
CPU-only
Intel Xeon
OpenMP runtime 参数调优
小型 LLM policy 训练
SFT + preference learning / DPO + 可选 RL
真实运行时间作为 reward
```

当前保留未来扩展方向：

```text
AMD 
ARM/鲲鹏
SW
多平台 architecture-aware tuning
RAG 经验库
LoRA policy
大小模型结合/调用api
```

------

## 3. OpenMP runtime action space

当前 action space 只包含 portable OpenMP runtime 标准变量，避免第一阶段引入 runtime-specific 参数，例如 `KMP_*`、`GOMP_*`。

推荐第一阶段 action space 为：

```text
OMP_NUM_THREADS
OMP_PROC_BIND
OMP_PLACES
OMP_SCHEDULE
OMP_DYNAMIC
OMP_WAIT_POLICY
OMP_MAX_ACTIVE_LEVELS
```

其中内部不直接使用绝对线程数，而使用 normalized action（待定）。

### 3.1 `OMP_NUM_THREADS`（待定 须先看数据集构建后的结果）

内部表示：

```text
0.25C
0.5C
1.0C
1.5C
2.0C
1NUMA
1Socket
```

其中：

```text
C = physical cores
1NUMA = 单个 NUMA node 上的核心数
1Socket = 单个 socket 上的核心数
```

harness 根据 Intel Xeon 平台 profile 将其转换为实际 `OMP_NUM_THREADS`。

------

### 3.2 `OMP_PROC_BIND`

候选值：

```text
false
close
spread
primary
```

------

### 3.3 `OMP_PLACES`

候选值：

```text
threads
cores
sockets
```

------

### 3.4 `OMP_SCHEDULE`

候选值：

```text
static
dynamic,1
dynamic,4
dynamic,16
dynamic,64
dynamic,256
guided,1
guided,4
guided,16
guided,64
guided,256
auto
```

需要注意：

```text
OMP_SCHEDULE 只有在代码中的循环使用 schedule(runtime) 时才真正生效。
```

因此 harness 需要记录：

```json
{
  "schedule_runtime_enabled": true
}
```

如果程序没有 `schedule(runtime)`，则：

```text
OMP_SCHEDULE 可以固定为默认值，
或者将其标记为 ineffective action。
```

------

### 3.5 `OMP_DYNAMIC`

候选值：

```text
false
true
```

第一阶段建议默认更多尝试 `false`，因为这样实验更可控。

------

### 3.6 `OMP_WAIT_POLICY`

候选值：

```text
active
passive
```

------

### 3.7 `OMP_MAX_ACTIVE_LEVELS`

候选值：

```text
1
2
4
```

如果程序没有 nested parallelism，则可以默认固定为：

```text
OMP_MAX_ACTIVE_LEVELS=1
```

以避免扩大无效搜索空间。

------

## 4. Platform profile

虽然当前只做 Intel Xeon，但仍然要使用结构化 platform profile，为后续扩展到 AMD、ARM/鲲鹏、SW 做准备。

当前 Intel Xeon platform profile 至少包含：

```json
{
  "platform_label": "Intel_Xeon",
  "vendor": "Intel",
  "isa": "x86_64",
  "cpu_model": "...",
  "num_sockets": 2,
  "numa_nodes": 2,
  "physical_cores": 64,
  "logical_cpus": 128,
  "threads_per_core": 2,
  "l1_cache_kb_per_core": "...",
  "l2_cache_kb_per_core": "...",
  "l3_cache_mb_total": "...",
  "compiler": "gcc-...",
  "openmp_runtime": "libgomp / libomp / libiomp",
  "os": "linux",
  "kernel": "...",
  "frequency_policy": "fixed / turbo / unknown"
}
```

当前阶段虽然只有 Intel Xeon，但模型输入仍然使用完整 profile，而不是只给一个标签。这是为了保证后续迁移到其他平台时输入格式不变。

------

## 5. Program features / Program summary（待定 尽量不进行人工标注）

模型不应直接依赖整段源码做所有判断。系统需要先通过 program summarizer 生成程序摘要。

程序摘要可以来自：

```text
静态源码分析
OpenMP pragma 扫描
benchmark metadata
轻量 profiling
人工标注，第一阶段可接受
```

第一阶段 program features 可以包括：

```json
{
  "num_parallel_regions": 3,
  "num_omp_for": 5,
  "has_reduction": true,
  "has_nested_parallel": false,
  "has_atomic": false,
  "has_critical": false,
  "schedule_runtime_enabled": true,
  "regular_loop": true,
  "irregular_access": false,
  "memory_bound": true,
  "compute_bound": false,
  "load_balance": "regular",
  "input_size": "large"
}
```

如果暂时无法自动提取所有特征，可以先用：

```text
benchmark metadata + 简单静态分析 + 人工标签
```

后续再增强为自动 program summarizer。

------

## 6. RL-compatible execution environment

系统将 OpenMP runtime tuning 建模为一个 RL-compatible environment。

### 6.1 State / Observation

当前状态包含：

```text
platform profile
program features
action schema
optional trial history
remaining budget
current best reward
```

示例：

```json
{
  "platform_profile": {...},
  "program_features": {...},
  "action_schema": {...},
  "trial_history": [
    {
      "action": {
        "threads": "1.0C",
        "proc_bind": "close",
        "places": "cores",
        "schedule": "static"
      },
      "reward": 0.02
    },
    {
      "action": {
        "threads": "0.5C",
        "proc_bind": "spread",
        "places": "cores",
        "schedule": "static"
      },
      "reward": 0.14
    }
  ],
  "remaining_budget": 5
}
```

------

### 6.2 Action

Action 是一个 normalized OpenMP runtime configuration。

示例：

```json
{
  "threads": "0.5C",
  "proc_bind": "spread",
  "places": "cores",
  "schedule": "static",
  "dynamic": false,
  "wait_policy": "passive",
  "max_active_levels": 1
}
```

------

### 6.3 Environment execution

harness 对 action 执行：

```text
1. action validation
2. normalized action materialization
3. 设置环境变量
4. 运行程序
5. correctness check
6. 多次性能测量
7. 计算 median runtime
8. 计算 reward
9. 写入 trial database
```

------

### 6.4 Reward

优化目标是最小化运行时间，但训练 reward 使用 correctness-gated log speedup。

定义：

```text
T_base = canonical baseline median runtime
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

这样：

```text
比 baseline 快：reward > 0
和 baseline 接近：reward ≈ 0
比 baseline 慢：reward < 0
```

------

## 7. Baseline

当前阶段建议使用 canonical OpenMP baseline：

```bash
export OMP_NUM_THREADS=<physical_cores>
export OMP_PROC_BIND=close
export OMP_PLACES=cores
export OMP_DYNAMIC=FALSE
export OMP_WAIT_POLICY=PASSIVE
export OMP_MAX_ACTIVE_LEVELS=1
```

如果程序支持 `schedule(runtime)`，则加入：

```bash
export OMP_SCHEDULE=static
```

该 baseline 用于 reward 计算。

此外实验报告中可以额外报告：

```text
runtime default baseline
expert baseline
random search baseline
grid / coarse search baseline
LLM SFT-only baseline
LLM SFT + DPO baseline
LLM SFT + DPO + RL baseline
```

------

## 8. Trial data collection

在 Intel Xeon 平台上，对多个 OpenMP benchmark 或应用 kernel 采集 trial 数据。

每条 trial 记录包含：

```json
{
  "program_id": "NPB_CG",
  "input_size": "class_C",
  "platform_id": "intel_xeon_2s_64c",
  "platform_profile": {...},
  "program_features": {...},
  "normalized_action": {
    "threads": "0.5C",
    "proc_bind": "spread",
    "places": "cores",
    "schedule": "static",
    "dynamic": false,
    "wait_policy": "passive",
    "max_active_levels": 1
  },
  "materialized_env": {
    "OMP_NUM_THREADS": "32",
    "OMP_PROC_BIND": "spread",
    "OMP_PLACES": "cores",
    "OMP_SCHEDULE": "static",
    "OMP_DYNAMIC": "FALSE",
    "OMP_WAIT_POLICY": "PASSIVE",
    "OMP_MAX_ACTIVE_LEVELS": "1"
  },
  "correct": true,
  "runtime_runs": [1.83, 1.81, 1.84],
  "runtime_median": 1.83,
  "runtime_std": 0.015,
  "baseline_runtime": 2.10,
  "speedup": 1.15,
  "reward": 0.1398
}
```

数据来源包括：

```text
canonical baseline
expert portfolio
random / stratified random sampling
coarse grid search
可选 LLM cold-start candidates
```

------

## 9. LLM policy training

模型侧训练一个小型 LLM policy。

可选模型：

```text
Qwen2.5-1.5B-Instruct
Qwen2.5-3B-Instruct
Qwen2.5-7B-Instruct
Qwen2.5-Coder-7B
```

当前第一阶段建议优先选择：

```text
Qwen2.5-1.5B 或 3B 做快速验证
Qwen2.5-7B 做主要实验
```

模型输入：

```text
program features / program summary
platform profile
action schema
optional trial history
```

模型输出：

```text
one or multiple normalized OpenMP actions
```

建议让模型输出 top-K 候选，而不是只输出一个 action。

例如：

```json
[
  {
    "threads": "0.5C",
    "proc_bind": "spread",
    "places": "cores",
    "schedule": "static",
    "dynamic": false,
    "wait_policy": "passive",
    "max_active_levels": 1
  },
  {
    "threads": "1.0C",
    "proc_bind": "close",
    "places": "cores",
    "schedule": "static",
    "dynamic": false,
    "wait_policy": "passive",
    "max_active_levels": 1
  }
]
```

------

## 10. Training stages

训练过程参考 Compiler-R1 的分阶段思想，但不直接机械照搬。

### 10.1 Stage 1: SFT

SFT 目标：

```text
让 LLM 学会输入格式
让 LLM 学会合法 action schema
让 LLM 学会输出 normalized OpenMP action
让 LLM 初步模仿高 reward 配置
```

SFT 数据构造方式：

```text
对每个 program-platform-input，
从真实 trial 中选择 top reward action 或 top-K actions，
构造 instruction-output 样本。
```

SFT 输入示例：

```json
{
  "instruction": "Recommend OpenMP runtime parameters for this program and platform.",
  "platform_profile": {...},
  "program_features": {...},
  "action_schema": {...}
}
```

SFT 输出示例：

```json
{
  "threads": "0.5C",
  "proc_bind": "spread",
  "places": "cores",
  "schedule": "static",
  "dynamic": false,
  "wait_policy": "passive",
  "max_active_levels": 1
}
```

------

### 10.2 Stage 2: Preference learning / DPO（待定 不确定是否要进行这个阶段）

DPO 或 preference learning 用于让模型偏好高 reward action。

构造方式：

```text
在同一个 program-platform-input 下，
选择高 reward action 作为 chosen，
选择低 reward action 作为 rejected。
```

样本：

```json
{
  "prompt": {
    "platform_profile": {...},
    "program_features": {...},
    "action_schema": {...}
  },
  "chosen": {
    "threads": "0.5C",
    "proc_bind": "spread",
    "places": "cores",
    "schedule": "static",
    "dynamic": false
  },
  "rejected": {
    "threads": "2.0C",
    "proc_bind": "false",
    "places": "threads",
    "schedule": "dynamic,1",
    "dynamic": true
  }
}
```

这一阶段比直接 RL 更稳定，能利用离线 trial 数据训练模型偏好。

------

### 10.3 Stage 3: Optional GRPO / PPO

当 harness、reward 和 action validation 稳定后，可以进一步使用 RL。

流程：

```text
1. 给定 program-platform state
2. LLM policy 生成 K 个 OpenMP actions
3. harness 真实执行这些 actions
4. 得到 correctness-gated log speedup
5. 用 GRPO/PPO 更新 policy
```

当前建议优先考虑 GRPO，因为同一个 state 下可以采样多个 action 并比较组内 reward。

PPO 可作为后续更完整 RL baseline，但工程复杂度更高。

------

## 11. Deployment / inference

部署时，对于一个新程序：

```text
1. program summarizer 提取 program features
2. 采集目标 Intel Xeon platform profile
3. LLM policy 根据 program features + platform profile + action schema 生成 top-K OpenMP actions
4. validator 过滤非法 action
5. harness 在目标机器上实测 top-K 或 top-N （待定 需要这一步的必要性）
6. 输出实测最快配置
7. 将 trial 写回数据库
```

这称为：

```text
small-budget local adaptation
```

这里的 local trial 指：

```text
在目标机器上真实运行少量候选 OpenMP 配置。
```