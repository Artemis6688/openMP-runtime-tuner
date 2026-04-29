root@342ded456a8a:/workspace# gcc --version
python3 --version
lscpu
numactl --hardware
gcc (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

Python 3.10.12
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         52 bits physical, 57 bits virtual
  Byte Order:            Little Endian
CPU(s):                  64
  On-line CPU(s) list:   0-63
Vendor ID:               GenuineIntel
  Model name:            Intel(R) Xeon(R) Gold 6430
    CPU family:          6
    Model:               143
    Thread(s) per core:  1
    Core(s) per socket:  32
    Socket(s):           2
    Stepping:            8
    CPU max MHz:         3400.0000
    CPU min MHz:         800.0000
    BogoMIPS:            4200.00
    Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 
                         clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtsc
                         p lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nons
                         top_tsc cpuid aperfmperf tsc_known_freq pni pclmulqdq dtes64 monitor ds_
                         cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2
                          x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_l
                         m abm 3dnowprefetch cpuid_fault epb cat_l3 cat_l2 cdp_l3 invpcid_single 
                         cdp_l2 ssbd mba ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriori
                         ty ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid 
                         cqm rdt_a avx512f avx512dq rdseed adx smap avx512ifma clflushopt clwb in
                         tel_pt avx512cd sha_ni avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves 
                         cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local avx512_bf16 wbnoinvd d
                         therm ida arat pln pts hwp hwp_act_window hwp_epp hwp_pkg_req avx512vbmi
                          umip pku ospke waitpkg avx512_vbmi2 gfni vaes vpclmulqdq avx512_vnni av
                         x512_bitalg tme avx512_vpopcntdq rdpid cldemote movdiri movdir64b md_cle
                         ar pconfig flush_l1d arch_capabilities
Virtualization features: 
  Virtualization:        VT-x
Caches (sum of all):     
  L1d:                   3 MiB (64 instances)
  L1i:                   2 MiB (64 instances)
  L2:                    128 MiB (64 instances)
  L3:                    120 MiB (2 instances)
NUMA:                    
  NUMA node(s):          2
  NUMA node0 CPU(s):     0-31
  NUMA node1 CPU(s):     32-63
Vulnerabilities:         
  Gather data sampling:  Not affected
  Itlb multihit:         Not affected
  L1tf:                  Not affected
  Mds:                   Not affected
  Meltdown:              Not affected
  Mmio stale data:       Not affected
  Retbleed:              Not affected
  Spec store bypass:     Mitigation; Speculative Store Bypass disabled via prctl and seccomp
  Spectre v1:            Mitigation; usercopy/swapgs barriers and __user pointer sanitization
  Spectre v2:            Mitigation; Enhanced / Automatic IBRS; IBPB conditional; RSB filling; PB
                         RSB-eIBRS SW sequence; BHI BHI_DIS_S
  Srbds:                 Not affected
  Tsx async abort:       Not affected
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
node 0 size: 257602 MB
node 0 free: 92728 MB
node 1 cpus: 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63
node 1 size: 258038 MB
node 1 free: 38274 MB
node distances:
node   0   1 
  0:  10  21 
  1:  21  10 
root@342ded456a8a:/workspace# lscpu --json
lscpu -e=CPU,CORE,SOCKET,NODE,ONLINE,MAXMHZ,MINMHZ,MHZ
taskset -pc $$
grep Cpus_allowed_list /proc/self/status
ldd --version
OMP_DISPLAY_ENV=VERBOSE ./omp_probe
{
   "lscpu": [
      {
         "field": "Architecture:",
         "data": "x86_64",
         "children": [
            {
               "field": "CPU op-mode(s):",
               "data": "32-bit, 64-bit"
            },{
               "field": "Address sizes:",
               "data": "52 bits physical, 57 bits virtual"
            },{
               "field": "Byte Order:",
               "data": "Little Endian"
            }
         ]
      },{
         "field": "CPU(s):",
         "data": "64",
         "children": [
            {
               "field": "On-line CPU(s) list:",
               "data": "0-63"
            }
         ]
      },{
         "field": "Vendor ID:",
         "data": "GenuineIntel",
         "children": [
            {
               "field": "Model name:",
               "data": "Intel(R) Xeon(R) Gold 6430",
               "children": [
                  {
                     "field": "CPU family:",
                     "data": "6"
                  },{
                     "field": "Model:",
                     "data": "143"
                  },{
                     "field": "Thread(s) per core:",
                     "data": "1"
                  },{
                     "field": "Core(s) per socket:",
                     "data": "32"
                  },{
                     "field": "Socket(s):",
                     "data": "2"
                  },{
                     "field": "Stepping:",
                     "data": "8"
                  },{
                     "field": "CPU max MHz:",
                     "data": "3400.0000"
                  },{
                     "field": "CPU min MHz:",
                     "data": "800.0000"
                  },{
                     "field": "BogoMIPS:",
                     "data": "4200.00"
                  },{
                     "field": "Flags:",
                     "data": "fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf tsc_known_freq pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault epb cat_l3 cat_l2 cdp_l3 invpcid_single cdp_l2 ssbd mba ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm rdt_a avx512f avx512dq rdseed adx smap avx512ifma clflushopt clwb intel_pt avx512cd sha_ni avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local avx512_bf16 wbnoinvd dtherm ida arat pln pts hwp hwp_act_window hwp_epp hwp_pkg_req avx512vbmi umip pku ospke waitpkg avx512_vbmi2 gfni vaes vpclmulqdq avx512_vnni avx512_bitalg tme avx512_vpopcntdq rdpid cldemote movdiri movdir64b md_clear pconfig flush_l1d arch_capabilities"
                  }
               ]
            }
         ]
      },{
         "field": "Virtualization features:",
         "data": null,
         "children": [
            {
               "field": "Virtualization:",
               "data": "VT-x"
            }
         ]
      },{
         "field": "Caches (sum of all):",
         "data": null,
         "children": [
            {
               "field": "L1d:",
               "data": "3 MiB (64 instances)"
            },{
               "field": "L1i:",
               "data": "2 MiB (64 instances)"
            },{
               "field": "L2:",
               "data": "128 MiB (64 instances)"
            },{
               "field": "L3:",
               "data": "120 MiB (2 instances)"
            }
         ]
      },{
         "field": "NUMA:",
         "data": null,
         "children": [
            {
               "field": "NUMA node(s):",
               "data": "2"
            },{
               "field": "NUMA node0 CPU(s):",
               "data": "0-31"
            },{
               "field": "NUMA node1 CPU(s):",
               "data": "32-63"
            }
         ]
      },{
         "field": "Vulnerabilities:",
         "data": null,
         "children": [
            {
               "field": "Gather data sampling:",
               "data": "Not affected"
            },{
               "field": "Itlb multihit:",
               "data": "Not affected"
            },{
               "field": "L1tf:",
               "data": "Not affected"
            },{
               "field": "Mds:",
               "data": "Not affected"
            },{
               "field": "Meltdown:",
               "data": "Not affected"
            },{
               "field": "Mmio stale data:",
               "data": "Not affected"
            },{
               "field": "Retbleed:",
               "data": "Not affected"
            },{
               "field": "Spec store bypass:",
               "data": "Mitigation; Speculative Store Bypass disabled via prctl and seccomp"
            },{
               "field": "Spectre v1:",
               "data": "Mitigation; usercopy/swapgs barriers and __user pointer sanitization"
            },{
               "field": "Spectre v2:",
               "data": "Mitigation; Enhanced / Automatic IBRS; IBPB conditional; RSB filling; PBRSB-eIBRS SW sequence; BHI BHI_DIS_S"
            },{
               "field": "Srbds:",
               "data": "Not affected"
            },{
               "field": "Tsx async abort:",
               "data": "Not affected"
            }
         ]
      }
   ]
}
CPU CORE SOCKET NODE ONLINE    MAXMHZ   MINMHZ      MHZ
  0    0      0    0    yes 3400.0000 800.0000  796.259
  1    1      0    0    yes 3400.0000 800.0000  795.543
  2    2      0    0    yes 3400.0000 800.0000  795.543
  3    3      0    0    yes 3400.0000 800.0000  789.228
  4    4      0    0    yes 3400.0000 800.0000  792.695
  5    5      0    0    yes 3400.0000 800.0000 2600.000
  6    6      0    0    yes 3400.0000 800.0000  789.934
  7    7      0    0    yes 3400.0000 800.0000  790.900
  8    8      0    0    yes 3400.0000 800.0000  785.362
  9    9      0    0    yes 3400.0000 800.0000  797.956
 10   10      0    0    yes 3400.0000 800.0000  799.330
 11   11      0    0    yes 3400.0000 800.0000  755.838
 12   12      0    0    yes 3400.0000 800.0000  792.272
 13   13      0    0    yes 3400.0000 800.0000  789.536
 14   14      0    0    yes 3400.0000 800.0000  755.000
 15   15      0    0    yes 3400.0000 800.0000  794.300
 16   16      0    0    yes 3400.0000 800.0000  797.318
 17   17      0    0    yes 3400.0000 800.0000  795.475
 18   18      0    0    yes 3400.0000 800.0000  794.554
 19   19      0    0    yes 3400.0000 800.0000  792.410
 20   20      0    0    yes 3400.0000 800.0000 1969.480
 21   21      0    0    yes 3400.0000 800.0000  798.689
 22   22      0    0    yes 3400.0000 800.0000  798.283
 23   23      0    0    yes 3400.0000 800.0000  798.713
 24   24      0    0    yes 3400.0000 800.0000  790.261
 25   25      0    0    yes 3400.0000 800.0000  798.144
 26   26      0    0    yes 3400.0000 800.0000  797.763
 27   27      0    0    yes 3400.0000 800.0000 1343.558
 28   28      0    0    yes 3400.0000 800.0000  799.641
 29   29      0    0    yes 3400.0000 800.0000  795.224
 30   30      0    0    yes 3400.0000 800.0000  791.309
 31   31      0    0    yes 3400.0000 800.0000  798.624
 32   32      1    1    yes 3400.0000 800.0000  789.200
 33   33      1    1    yes 3400.0000 800.0000  789.488
 34   34      1    1    yes 3400.0000 800.0000  793.908
 35   35      1    1    yes 3400.0000 800.0000 2036.271
 36   36      1    1    yes 3400.0000 800.0000  762.522
 37   37      1    1    yes 3400.0000 800.0000  790.914
 38   38      1    1    yes 3400.0000 800.0000  797.283
 39   39      1    1    yes 3400.0000 800.0000  775.726
 40   40      1    1    yes 3400.0000 800.0000  792.171
 41   41      1    1    yes 3400.0000 800.0000  790.533
 42   42      1    1    yes 3400.0000 800.0000 2600.000
 43   43      1    1    yes 3400.0000 800.0000  789.239
 44   44      1    1    yes 3400.0000 800.0000  787.373
 45   45      1    1    yes 3400.0000 800.0000  788.243
 46   46      1    1    yes 3400.0000 800.0000  791.903
 47   47      1    1    yes 3400.0000 800.0000  771.566
 48   48      1    1    yes 3400.0000 800.0000  764.309
 49   49      1    1    yes 3400.0000 800.0000  784.702
 50   50      1    1    yes 3400.0000 800.0000  763.321
 51   51      1    1    yes 3400.0000 800.0000  790.843
 52   52      1    1    yes 3400.0000 800.0000  794.897
 53   53      1    1    yes 3400.0000 800.0000  797.634
 54   54      1    1    yes 3400.0000 800.0000  791.700
 55   55      1    1    yes 3400.0000 800.0000  795.506
 56   56      1    1    yes 3400.0000 800.0000  796.762
 57   57      1    1    yes 3400.0000 800.0000  792.719
 58   58      1    1    yes 3400.0000 800.0000  795.691
 59   59      1    1    yes 3400.0000 800.0000  789.395
 60   60      1    1    yes 3400.0000 800.0000  799.487
 61   61      1    1    yes 3400.0000 800.0000  799.139
 62   62      1    1    yes 3400.0000 800.0000  799.185
 63   63      1    1    yes 3400.0000 800.0000  789.981
pid 9's current affinity list: 0-63
Cpus_allowed_list:      0-63
ldd (Ubuntu GLIBC 2.35-0ubuntu3.10) 2.35
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
bash: ./omp_probe: No such file or directory
root@342ded456a8a:/workspace# 