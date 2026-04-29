# Intel Xeon 平台信息提取汇总

**用途**：为 PolyBench/C OpenMP runtime tuning v0 数据集构建提供平台 profile、action space 生成依据、baseline 设置依据和后续交接说明。

**信息来源**：

```text
第一阶段md文档/x86intel架构具体信息.md
用户后续补充的 omp_probe / ldd / OMP_DISPLAY_ENV / uname 输出
```

**提取原则**：只使用已给出的命令输出；未出现或未成功采集的信息标记为“待补充”，不自行编造。

------

## 1. 已采集命令

当前已经包含以下命令输出：

```bash
gcc --version
python3 --version
lscpu
numactl --hardware
lscpu --json
lscpu -e=CPU,CORE,SOCKET,NODE,ONLINE,MAXMHZ,MINMHZ,MHZ
taskset -pc $$
grep Cpus_allowed_list /proc/self/status
ldd --version
OMP_DISPLAY_ENV=VERBOSE ./omp_probe
gcc -O2 -fopenmp /workspace/omp_probe.c -o /workspace/omp_probe_gcc
ldd /workspace/omp_probe_gcc
OMP_DISPLAY_ENV=VERBOSE /workspace/omp_probe_gcc
uname -a
env | sort | grep -E '^(OMP|GOMP|KMP|MKL|OPENBLAS|SLURM|PBS|LSB|LD_LIBRARY_PATH)=' || true
dpkg -S /lib/x86_64-linux-gnu/libgomp.so.1
dpkg -l | grep -E '^ii\s+libgomp|^ii\s+gcc|^ii\s+g\+\+'
```

原始文件中 `OMP_DISPLAY_ENV=VERBOSE ./omp_probe` 曾未成功执行，原因是当前目录下没有 `./omp_probe`：

```text
bash: ./omp_probe: No such file or directory
```

用户后续已经重新编译并运行 `/workspace/omp_probe_gcc`，因此当前已经可以确认 OpenMP runtime 链接库、`OMP_DISPLAY_ENV` 输出和 `omp_get_*` 探测结果。

------

## 2. 编译与运行环境

### 2.1 GCC

```json
{
  "compiler": {
    "name": "gcc",
    "version": "11.4.0",
    "package": "Ubuntu 11.4.0-1ubuntu1~22.04.3",
    "dpkg_packages": {
      "gcc": "4:11.2.0-1ubuntu1",
      "gcc-11": "11.4.0-1ubuntu1~22.04.3",
      "gcc-11-base:amd64": "11.4.0-1ubuntu1~22.04.3",
      "gcc-12-base:amd64": "12.3.0-1ubuntu1~22.04.3",
      "g++": "4:11.2.0-1ubuntu1",
      "g++-11": "11.4.0-1ubuntu1~22.04.3"
    }
  }
}
```

原始输出：

```text
gcc (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0
```

补充 `dpkg -l` 输出中与 GCC/G++ 相关的包版本：

```text
ii  g++                         4:11.2.0-1ubuntu1                       amd64        GNU C++ compiler
ii  g++-11                      11.4.0-1ubuntu1~22.04.3                 amd64        GNU C++ compiler
ii  gcc                         4:11.2.0-1ubuntu1                       amd64        GNU C compiler
ii  gcc-11                      11.4.0-1ubuntu1~22.04.3                 amd64        GNU C compiler
ii  gcc-11-base:amd64           11.4.0-1ubuntu1~22.04.3                 amd64        GCC, the GNU Compiler Collection (base package)
ii  gcc-12-base:amd64           12.3.0-1ubuntu1~22.04.3                 amd64        GCC, the GNU Compiler Collection (base package)
```

### 2.2 Python

```json
{
  "python": {
    "version": "3.10.12"
  }
}
```

原始输出：

```text
Python 3.10.12
```

### 2.3 glibc / ldd

```json
{
  "glibc": {
    "version": "2.35",
    "package": "Ubuntu GLIBC 2.35-0ubuntu3.10"
  }
}
```

原始输出：

```text
ldd (Ubuntu GLIBC 2.35-0ubuntu3.10) 2.35
```

### 2.4 工作环境提示

原文件命令行提示符显示当前环境为容器内 `/workspace`：

```text
root@342ded456a8a:/workspace#
```

该字段只能说明命令是在容器内执行，不能单独作为完整 OS profile。

### 2.5 Kernel / uname

```json
{
  "os": "linux",
  "kernel": "5.4.0-216-generic",
  "uname": "Linux 342ded456a8a 5.4.0-216-generic #236-Ubuntu SMP Fri Apr 11 19:53:21 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux"
}
```

原始输出：

```text
Linux 342ded456a8a 5.4.0-216-generic #236-Ubuntu SMP Fri Apr 11 19:53:21 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

------

## 3. CPU 基础信息

### 3.1 架构与字节序

```json
{
  "isa": "x86_64",
  "cpu_op_modes": ["32-bit", "64-bit"],
  "address_sizes": {
    "physical": "52 bits",
    "virtual": "57 bits"
  },
  "byte_order": "Little Endian"
}
```

原始输出：

```text
Architecture:            x86_64
CPU op-mode(s):        32-bit, 64-bit
Address sizes:         52 bits physical, 57 bits virtual
Byte Order:            Little Endian
```

### 3.2 CPU 型号

```json
{
  "vendor_id": "GenuineIntel",
  "cpu_model_name": "Intel(R) Xeon(R) Gold 6430",
  "cpu_family": 6,
  "model": 143,
  "stepping": 8,
  "bogomips": 4200.00
}
```

原始输出：

```text
Vendor ID:               GenuineIntel
Model name:            Intel(R) Xeon(R) Gold 6430
CPU family:          6
Model:               143
Stepping:            8
BogoMIPS:            4200.00
```

------

## 4. CPU 拓扑信息

### 4.1 原始拓扑字段

```json
{
  "logical_cpus": 64,
  "online_cpu_list": "0-63",
  "threads_per_core": 1,
  "cores_per_socket": 32,
  "sockets": 2,
  "numa_nodes": 2,
  "numa_node_cpus": {
    "0": "0-31",
    "1": "32-63"
  }
}
```

原始输出：

```text
CPU(s):                  64
On-line CPU(s) list:   0-63
Thread(s) per core:  1
Core(s) per socket:  32
Socket(s):           2
NUMA node(s):          2
NUMA node0 CPU(s):     0-31
NUMA node1 CPU(s):     32-63
```

### 4.2 由原始字段直接得到的派生信息

这些字段是由上面的原始拓扑信息直接计算得到：

```json
{
  "physical_cores": 64,
  "logical_cpus": 64,
  "cores_per_numa_node": 32,
  "hyper_threading_visible": false
}
```

计算依据：

```text
physical_cores = cores_per_socket * sockets = 32 * 2 = 64
logical_cpus = 64
threads_per_core = 1
cores_per_numa_node = 64 / 2 = 32
```

注意：这里只能说明当前环境中可见 `Thread(s) per core = 1`。不能进一步判断宿主机是否在 BIOS 或调度层面关闭了超线程。

### 4.3 CPU / Core / Socket / NUMA 映射

原文件中 `lscpu -e=CPU,CORE,SOCKET,NODE,ONLINE,MAXMHZ,MINMHZ,MHZ` 显示：

```text
CPU 0-31  -> SOCKET 0, NODE 0, ONLINE yes
CPU 32-63 -> SOCKET 1, NODE 1, ONLINE yes
```

且每个 CPU 的 `CORE` 编号与 `CPU` 编号一致：

```text
CPU 0  CORE 0
CPU 1  CORE 1
...
CPU 63 CORE 63
```

这与 `Thread(s) per core: 1` 一致。

------

## 5. NUMA 与内存信息

### 5.1 NUMA CPU 分布

```json
{
  "numa_nodes": 2,
  "numa_node_cpus": {
    "0": "0-31",
    "1": "32-63"
  }
}
```

### 5.2 NUMA 内存

```json
{
  "numa_node_memory_mb": {
    "0": {
      "size": 257602,
      "free_at_collection_time": 92728
    },
    "1": {
      "size": 258038,
      "free_at_collection_time": 38274
    }
  }
}
```

原始输出：

```text
node 0 size: 257602 MB
node 0 free: 92728 MB
node 1 size: 258038 MB
node 1 free: 38274 MB
```

说明：`free` 是采集时刻的瞬时值，后续实验报告中应作为采集记录，不应当作稳定硬件属性。

### 5.3 NUMA 距离

```json
{
  "numa_distance": {
    "0_to_0": 10,
    "0_to_1": 21,
    "1_to_0": 21,
    "1_to_1": 10
  }
}
```

原始输出：

```text
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

------

## 6. CPU 频率信息

### 6.1 硬件频率范围

```json
{
  "cpu_min_mhz": 800.0000,
  "cpu_max_mhz": 3400.0000
}
```

原始输出：

```text
CPU max MHz:         3400.0000
CPU min MHz:         800.0000
```

### 6.2 采集时刻的 per-CPU MHz

`lscpu -e` 输出中每个 CPU 都有采集时刻的 `MHZ` 字段，大多数 CPU 约在 755-800 MHz 附近，少数 CPU 在采集时刻显示为更高频率，例如：

```text
CPU 5  MHZ 2600.000
CPU 20 MHZ 1969.480
CPU 27 MHZ 1343.558
CPU 35 MHZ 2036.271
CPU 42 MHZ 2600.000
```

说明：这些 `MHZ` 是采集瞬间的观测值，不能据此确认频率 governor、turbo 状态或实验期间固定频率策略。

当前应记录：

```json
{
  "frequency_policy": "unknown",
  "turbo": "unknown"
}
```

------

## 7. Cache 信息

### 7.1 原始 cache 字段

```json
{
  "l1d_cache_total": "3 MiB",
  "l1d_cache_instances": 64,
  "l1i_cache_total": "2 MiB",
  "l1i_cache_instances": 64,
  "l2_cache_total": "128 MiB",
  "l2_cache_instances": 64,
  "l3_cache_total": "120 MiB",
  "l3_cache_instances": 2
}
```

原始输出：

```text
L1d:                   3 MiB (64 instances)
L1i:                   2 MiB (64 instances)
L2:                    128 MiB (64 instances)
L3:                    120 MiB (2 instances)
```

### 7.2 由原始字段直接得到的派生 cache 信息

```json
{
  "l1d_cache_per_instance": "48 KiB",
  "l1i_cache_per_instance": "32 KiB",
  "l2_cache_per_instance": "2 MiB",
  "l3_cache_per_instance": "60 MiB"
}
```

计算依据：

```text
L1d: 3 MiB / 64 = 48 KiB
L1i: 2 MiB / 64 = 32 KiB
L2: 128 MiB / 64 = 2 MiB
L3: 120 MiB / 2 = 60 MiB
```

------

## 8. CPU 特性中对本项目较相关的信息

原始 `Flags` 很长。对 PolyBench/C 数值 kernel 和后续程序摘要可能较相关的特性包括：

```text
sse
sse2
sse4_1
sse4_2
avx
avx2
fma
avx512f
avx512dq
avx512cd
avx512bw
avx512vl
avx512ifma
avx512_bf16
avx512vbmi
avx512_vbmi2
avx512_vnni
avx512_bitalg
avx512_vpopcntdq
gfni
vaes
vpclmulqdq
```

说明：当前第一阶段调优的是 OpenMP runtime 参数，不做 compiler flag tuning，因此这些 ISA 特性主要作为 platform profile 或程序摘要背景信息，不应被用于改变 v0 的编译优化策略。

------

## 9. CPU 可用性与 cpuset

### 9.1 taskset

```json
{
  "taskset_current_affinity": "0-63"
}
```

原始输出：

```text
pid 9's current affinity list: 0-63
```

### 9.2 Cpus_allowed_list

```json
{
  "cpus_allowed_list": "0-63"
}
```

原始输出：

```text
Cpus_allowed_list:      0-63
```

### 9.3 对实验的含义

当前容器进程可见并允许使用 CPU `0-63`，即 64 个 CPU。

因此当前 v0 的 `OMP_NUM_THREADS` action 不应超过 64，除非后续重新采集到不同的 cpuset 或宿主环境信息。

------

## 10. OpenMP runtime 信息状态

### 10.1 OpenMP 编译器与链接库

当前使用 `gcc -O2 -fopenmp` 编译 OpenMP probe。

`ldd /workspace/omp_probe_gcc` 输出显示链接到 `libgomp.so.1`：

```text
linux-vdso.so.1 (0x00007ffce69bf000)
libgomp.so.1 => /lib/x86_64-linux-gnu/libgomp.so.1 (0x00007fb0b23b5000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fb0b218c000)
/lib64/ld-linux-x86-64.so.2 (0x00007fb0b2409000)
```

可确认：

```json
{
  "compiler_for_openmp": "gcc 11.4.0",
  "openmp_runtime": {
    "name": "libgomp",
    "library": "/lib/x86_64-linux-gnu/libgomp.so.1",
    "package": "libgomp1:amd64",
    "exact_package_version": "12.3.0-1ubuntu1~22.04.3"
  }
}
```

补充包查询输出：

```text
dpkg-query: no path found matching pattern /lib/x86_64-linux-gnu/libgomp.so.1
ii  libgomp1:amd64              12.3.0-1ubuntu1~22.04.3                 amd64        GCC OpenMP (GOMP) support library
```

说明：

```text
ldd 输出确认 OpenMP runtime 链接库为 /lib/x86_64-linux-gnu/libgomp.so.1。
dpkg -S 直接查询该路径没有匹配到所属包。
dpkg -l 确认当前安装的 GCC OpenMP runtime 包为 libgomp1:amd64，版本为 12.3.0-1ubuntu1~22.04.3。
当前 GCC 编译器版本是 11.4.0，但 libgomp1 runtime 包版本是 12.3.0-1ubuntu1~22.04.3；二者应分别记录。
```

### 10.2 OpenMP 标准宏与 runtime probe

```json
{
  "_OPENMP": 201511,
  "openmp_standard_from_macro": "OpenMP 4.5",
  "omp_get_num_procs": 64,
  "omp_get_max_threads": 64,
  "omp_get_dynamic": 0,
  "omp_get_nested": 0,
  "omp_get_max_active_levels": 1,
  "actual_parallel_threads": 64
}
```

原始输出：

```text
_OPENMP=201511
omp_get_num_procs=64
omp_get_max_threads=64
omp_get_dynamic=0
omp_get_nested=0
omp_get_max_active_levels=1
actual_parallel_threads=64
```

### 10.3 `OMP_DISPLAY_ENV=VERBOSE` 输出

补充采集时的 OpenMP 环境显示如下：

```json
{
  "_OPENMP": "201511",
  "OMP_DYNAMIC": "FALSE",
  "OMP_NESTED": "FALSE",
  "OMP_NUM_THREADS": "64",
  "OMP_SCHEDULE": "DYNAMIC",
  "OMP_PROC_BIND": "FALSE",
  "OMP_PLACES": "",
  "OMP_STACKSIZE": "0",
  "OMP_WAIT_POLICY": "PASSIVE",
  "OMP_THREAD_LIMIT": "4294967295",
  "OMP_MAX_ACTIVE_LEVELS": "1",
  "OMP_NUM_TEAMS": "0",
  "OMP_TEAMS_THREAD_LIMIT": "0",
  "OMP_CANCELLATION": "FALSE",
  "OMP_DEFAULT_DEVICE": "0",
  "OMP_MAX_TASK_PRIORITY": "0",
  "OMP_DISPLAY_AFFINITY": "FALSE",
  "OMP_AFFINITY_FORMAT": "level %L thread %i affinity %A",
  "OMP_ALLOCATOR": "omp_default_mem_alloc",
  "OMP_TARGET_OFFLOAD": "DEFAULT",
  "GOMP_CPU_AFFINITY": "",
  "GOMP_STACKSIZE": "0",
  "GOMP_SPINCOUNT": "300000"
}
```

对本项目的直接含义：

```text
默认最大线程数与当前可见 CPU 数一致，均为 64。
默认 OMP_PROC_BIND 为 FALSE，正式 trial 中应由 action 显式设置。
默认 OMP_PLACES 为空，正式 trial 中应由 action 显式设置。
默认 OMP_SCHEDULE 显示为 DYNAMIC，canonical baseline 中应显式设置为 static。
OMP_MAX_ACTIVE_LEVELS 默认为 1，符合当前 v0 不引入 nested parallelism 的设定。
```

### 10.4 相关环境变量

用户补充的命令中包含：

```bash
env | sort | grep -E '^(OMP|GOMP|KMP|MKL|OPENBLAS|SLURM|PBS|LSB|LD_LIBRARY_PATH)=' || true
```

在补充输出中未看到该命令产生匹配行。为避免误判，当前记录为：

```json
{
  "relevant_env_vars_observed_in_paste": "no_matching_lines_observed",
  "strict_env_capture": "待补充，可在正式采集脚本中保存到文件"
}
```

------

## 11. 可直接用于 `platform_profile.json` 的草稿

以下 JSON 只包含当前文件已确认或由当前文件直接计算出的字段：

```json
{
  "platform_id": "intel_xeon_gold_6430_container_v0",
  "platform_label": "Intel_Xeon",
  "vendor": "Intel",
  "vendor_id": "GenuineIntel",
  "isa": "x86_64",
  "cpu_model": "Intel(R) Xeon(R) Gold 6430",
  "cpu_family": 6,
  "model": 143,
  "stepping": 8,
  "num_sockets": 2,
  "numa_nodes": 2,
  "physical_cores": 64,
  "logical_cpus": 64,
  "threads_per_core": 1,
  "cores_per_socket": 32,
  "cores_per_numa_node": 32,
  "online_cpus": "0-63",
  "available_cpus": "0-63",
  "cpus_allowed_list": "0-63",
  "numa_node_cpus": {
    "0": "0-31",
    "1": "32-63"
  },
  "numa_node_memory_mb": {
    "0": {
      "size": 257602,
      "free_at_collection_time": 92728
    },
    "1": {
      "size": 258038,
      "free_at_collection_time": 38274
    }
  },
  "numa_distance": {
    "0_to_0": 10,
    "0_to_1": 21,
    "1_to_0": 21,
    "1_to_1": 10
  },
  "cache": {
    "l1d_total": "3 MiB",
    "l1d_instances": 64,
    "l1i_total": "2 MiB",
    "l1i_instances": 64,
    "l2_total": "128 MiB",
    "l2_instances": 64,
    "l3_total": "120 MiB",
    "l3_instances": 2
  },
  "frequency": {
    "cpu_min_mhz": 800.0,
    "cpu_max_mhz": 3400.0,
    "frequency_policy": "unknown",
    "turbo": "unknown"
  },
  "compiler": {
    "name": "gcc",
    "version": "11.4.0",
    "package": "Ubuntu 11.4.0-1ubuntu1~22.04.3",
    "dpkg_packages": {
      "gcc": "4:11.2.0-1ubuntu1",
      "gcc-11": "11.4.0-1ubuntu1~22.04.3",
      "gcc-11-base:amd64": "11.4.0-1ubuntu1~22.04.3",
      "gcc-12-base:amd64": "12.3.0-1ubuntu1~22.04.3",
      "g++": "4:11.2.0-1ubuntu1",
      "g++-11": "11.4.0-1ubuntu1~22.04.3"
    }
  },
  "python": {
    "version": "3.10.12"
  },
  "glibc": {
    "version": "2.35",
    "package": "Ubuntu GLIBC 2.35-0ubuntu3.10"
  },
  "os": {
    "name": "linux",
    "kernel": "5.4.0-216-generic",
    "uname": "Linux 342ded456a8a 5.4.0-216-generic #236-Ubuntu SMP Fri Apr 11 19:53:21 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux"
  },
  "openmp_runtime": {
    "name": "libgomp",
    "library": "/lib/x86_64-linux-gnu/libgomp.so.1",
    "package": "libgomp1:amd64",
    "exact_package_version": "12.3.0-1ubuntu1~22.04.3",
    "openmp_macro": 201511,
    "openmp_standard_from_macro": "OpenMP 4.5",
    "omp_get_num_procs": 64,
    "omp_get_max_threads": 64,
    "omp_get_dynamic": 0,
    "omp_get_nested": 0,
    "omp_get_max_active_levels": 1,
    "actual_parallel_threads": 64
  },
  "openmp_display_env_observed": {
    "OMP_DYNAMIC": "FALSE",
    "OMP_NESTED": "FALSE",
    "OMP_NUM_THREADS": "64",
    "OMP_SCHEDULE": "DYNAMIC",
    "OMP_PROC_BIND": "FALSE",
    "OMP_PLACES": "",
    "OMP_WAIT_POLICY": "PASSIVE",
    "OMP_MAX_ACTIVE_LEVELS": "1",
    "GOMP_CPU_AFFINITY": "",
    "GOMP_SPINCOUNT": "300000"
  }
}
```

------

## 12. 对 OpenMP runtime action space 的影响

### 12.1 `OMP_NUM_THREADS` 候选

根据当前已确认字段：

```text
physical_cores = 64
logical_cpus = 64
cores_per_socket = 32
cores_per_numa_node = 32
available_cpus = 0-63
```

当前 v0 建议候选集合：

```text
1
2
4
8
16
32
48
64
```

说明：

```text
32 = 单 socket / 单 NUMA node 的核心数
64 = 当前容器可用的全部 CPU 数
48 = 介于单 NUMA 与全机之间的补充采样点
```

当前不建议加入：

```text
96
128
```

原因：当前文件中可见 `CPU(s): 64`，`Cpus_allowed_list: 0-63`，没有可使用 96 或 128 个 CPU 的证据。

### 12.2 其他 OpenMP 标准变量候选

这些变量来自项目 v0 action space 设计，不由平台文件新增：

```json
{
  "OMP_PROC_BIND": ["false", "close", "spread"],
  "OMP_PLACES": ["threads", "cores", "sockets"],
  "OMP_SCHEDULE": ["static", "dynamic,1", "dynamic,16", "dynamic,64", "guided,16", "guided,64", "auto"],
  "OMP_DYNAMIC": ["FALSE", "TRUE"],
  "OMP_WAIT_POLICY": ["PASSIVE", "ACTIVE"],
  "OMP_MAX_ACTIVE_LEVELS": ["1"]
}
```

说明：当前项目 v0 不引入 nested parallelism，因此 `OMP_MAX_ACTIVE_LEVELS` 固定为 `1`。

------

## 13. 当前 canonical baseline 建议

基于当前文件中确认的 `available_cpus = 0-63`、`physical_cores = 64`：

```bash
export OMP_NUM_THREADS=64
export OMP_PROC_BIND=close
export OMP_PLACES=cores
export OMP_SCHEDULE=static
export OMP_DYNAMIC=FALSE
export OMP_WAIT_POLICY=PASSIVE
export OMP_MAX_ACTIVE_LEVELS=1
```

说明：该 baseline 是数据集构建文档中的 canonical OpenMP baseline 在当前平台上的 materialized 版本。补充 probe 已确认默认 `OMP_NUM_THREADS=64`，但正式 trial 仍应由 harness 显式设置所有 baseline 环境变量，避免继承 shell 或容器环境造成漂移。

------

## 14. 仍需补充采集的信息

当前已经补充了 `libgomp` 链接库、`libgomp1` 精确包版本、`_OPENMP` 宏、`omp_get_*`、`OMP_DISPLAY_ENV=VERBOSE` 和 `uname -a`。仍建议在正式 trial dataset 前补充以下信息：

```text
env 中已有 OMP/GOMP/KMP/MKL/OPENBLAS 变量的严格文件化记录
是否存在任务调度器变量，如 SLURM/PBS/LSB
cpupower frequency-info 或可读 governor/turbo 状态
```

### 14.1 建议补充命令

以下命令只做读取，不修改硬件状态，不需要重启服务器：

```bash
env | sort | grep -E '^(OMP|GOMP|KMP|MKL|OPENBLAS|SLURM|PBS|LSB|LD_LIBRARY_PATH)=' || true
```

如需补充频率策略，只建议执行只读命令：

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null || true
cat /sys/devices/system/cpu/intel_pstate/no_turbo 2>/dev/null || true
```

不要执行会修改频率、turbo、governor 或系统状态的命令。

------

## 15. 交接结论

当前平台可视为：

```text
Intel Xeon Gold 6430
x86_64
2 sockets
2 NUMA nodes
64 visible physical cores
64 visible logical CPUs
1 thread per core
CPU 0-31 on socket/node 0
CPU 32-63 on socket/node 1
GCC 11.4.0
OpenMP runtime: libgomp (/lib/x86_64-linux-gnu/libgomp.so.1)
OpenMP runtime package: libgomp1:amd64 12.3.0-1ubuntu1~22.04.3
_OPENMP macro: 201511 (OpenMP 4.5)
omp_get_num_procs: 64
omp_get_max_threads: 64
actual_parallel_threads in probe: 64
Python 3.10.12
glibc 2.35
Linux kernel 5.4.0-216-generic
```

对 PolyBench/C OpenMP runtime tuning v0，当前最关键的可执行结论是：

```text
OMP_NUM_THREADS 最大候选值应先按 64 处理。
canonical baseline 的 OMP_NUM_THREADS 应先设为 64。
不能把 128 线程作为当前环境的合法候选，除非后续重新采集证明可用。
OpenMP runtime 已通过 ldd 确认为 libgomp，当前安装的 libgomp1 包版本为 12.3.0-1ubuntu1~22.04.3。
默认 OMP_SCHEDULE 显示为 DYNAMIC，正式 baseline 和 trial 必须显式设置 OMP_SCHEDULE。
```
