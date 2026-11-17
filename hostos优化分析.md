# 宿主机(ECS)配置分析与沙箱性能优化项

> 采集时间:2026-11-12 · 对象:x86 ECS 10.30.33.139(AMD EPYC 9T25)、arm ECS 10.30.33.67(鲲鹏 920)
> 视角:针对沙箱 ppt_replay harness(进程编排型,32 命令生成 PPTX)与 workload.py(fib/memory/fileio/fork/npm)做宿主侧优化。

## 一、宿主机概况(实测)

| 维度 | x86 (139) | arm (67) |
|---|---|---|
| CPU | AMD EPYC 9T25 128C×2S×2T = 512 逻辑核 | 鲲鹏 920 7280Z 80C×2S = 160 核(无 SMT) |
| NUMA | 2 节点,距离 32 | 4 节点,距离 12/35~40 |
| 内存 | 1.5 TiB,无 swap | 1.0 TiB,无 swap |
| cpufreq | performance | performance |
| cpuidle | acpi_idle / menu | none(无驱动) |
| THP | [always] | [always] |
| 调度器 | EEVDF(6.6.0-72,CFS 已替换) | EEVDF(6.6.0-72,CFS 已替换) |
| 宿主 HZ | 1000(1ms tick) | 250(4ms tick) |
| clocksource | tsc | arch_sys_counter |
| irqbalance | active(默认 IRQ 亲和=全核) | 同左 |
| cmdline 关键 | nohz=off, mitigations=off, nmi_watchdog=1 | nohz=off, nospectre_bhb, iommu.passthrough=1, kvm.force_wfi_trap=1 |
| 沙箱规格 | 4 vCPU / 8GB(conductor: memory 8GiB capacity/allocatable) | 同左 |
| runner cpuset | cpuset.cpus=8-511, cpuset.mems=空 | cpuset.cpus=8-143, cpuset.mems=空 |

## 二、可优化项(按优先级)

### 1.【高】vCPU / 内存未按 NUMA 绑核绑内存
- 含义:沙箱 vCPU 线程与 8GB 内存在 2~4 个 NUMA 节点间自由漂移;`cpuset.mems` 为空 → 内存可落任意远端节点。
- 作用:4 vCPU guest 跨节点漂移 → 远端内存访问延迟 2~3×;vCPU 迁移丢 cache/TLB;调度抖动。**是 arm harness 逐轮 std 3.5~4.3%「宿主漂移」的头号嫌疑**。
- 优化方向:每沙箱 `cpuset.cpus` 固定到单节点内一段核、`cpuset.mems` 绑定同节点;或 cloud-hypervisor `--cpus`+numactl membind。
- 预期:显著压方差,mean 或小幅下降。

### 2.【高】halt_poll_ns 收缩关闭
- 当前:x86 200us / arm 500us;`halt_poll_ns_grow=2 / grow_start=10000 / shrink=0`(只增不减)。
- 含义:vCPU halt 后先忙等 halt_poll_ns 再让核,用空转换短 idle 唤醒低延迟。
- 作用:harness 短命令、npm node 子进程、fork 64 子进程均短 idle-wakeup 密集。
- 优化方向:宿主 CPU 几乎全空,空转代价≈0;x86 试提 300~500us,并打开 shrink 自适应回落。

### 3.【中】无 isolcpus / nohz_full(tick 抖动,x86 尤甚)
- 含义:cmdline `nohz=off`,无 isolcpus/nohz_full/rcu_nocbs,全核被周期 tick 打断。宿主 `CONFIG_NO_HZ_FULL=y` 已编译,只是被 `nohz=off` 关掉 → **改 cmdline 即可,不用重编内核**。
- 作用:宿主 tick/RCU/中断周期性抢占 vCPU 线程 → jitter。x86 宿主 `HZ=1000`(1ms tick)、arm `HZ=250`(4ms),**x86 tick 压力是 arm 的 4 倍**,隔离收益 x86 更明显。
- 优化方向:划专用核 `isolcpus + nohz_full + rcu_nocbs`(去掉 `nohz=off`),housekeeping 绑 0-7;需改 conductor 核分配逻辑。

### 4.【中】tdp_mmu = N(x86)
- 含义:新一代 TDP MMU 关闭,走 legacy MMU。
- 作用:memory 256MB 分配、npm 4318 文件 → 大量 EPT 缺页;legacy MMU 慢且多 vCPU 争 mmu_lock。
- 优化方向:评估 `tdp_mmu=Y`(上游默认开,较安全)。

### 5.【中】kvm_amd.avic = N(x86)
- 含义:AMD 硬件虚拟中断加速关闭,force_avic=N。
- 作用:guest 每次 timer/IPI/设备中断 trap 进 VMM → vmexit 开销;fork/exec/soffice/node 中断密集。
- 优化方向:评估 `avic=1`。⚠️ 与 nested=1、sev=Y 有兼容性约束(很可能正因此被禁),先验证稳定性;冲突则保持关,从 halt_poll/调度侧补。

### 6.【低】numa_balancing=1 与绑核冲突
- 含义:宿主 NUMA 自动平衡开,周期性迁移「放错节点」的页。
- 作用:绑核后反而来回迁 guest 页制造 churn。
- 优化方向:绑核落地后置 `kernel.numa_balancing=0`。

### 7.【低】arm twed_enable = N
- 含义:TWED 控制 guest WFE 自旋是否 trap 进 VMM 延迟。
- 作用:不 trap→零延迟但耗宿主核;trap→省核有延迟。
- 优化方向:宿主 CPU 富余,默认保持;仅当 guest WFE 空转明显才考虑开+小 twedel。

### 8.【低】guest 内存用 1GB 大页(x86)
- 含义:THP 已 2MB always;x86 支持 pdpe1gb,静态大页未预留(HugePages_Total=0)。
- 作用:2MB 已够;1GB 再降一级 EPT 层级。
- 优化方向:收益有限,需改 cloud-hypervisor 内存后端,优先级低。

### 9.【低】C-state
- x86 acpi_idle+menu 深 C-state 增唤醒延迟;arm cpuidle=none 已常活跃低延迟。
- 优化方向:x86 可试 `processor.max_cstate=1`;有 halt_poll 后收益被掩盖。

### 10.【低·不纳入】cluster 调度(共享 L2 调度域)
- 概念:调度域层级 `SMT → MC → CLUSTER → DIE → PKG → NUMA` 中的 CLUSTER(CLS) 级(`CONFIG_SCHED_CLUSTER`),专门建模「一小组核共享同一块 L2」,用于「唤醒放置」时在「同 cluster 缓存亲和」与「跨 cluster 摊薄 L2 容量」之间取舍;源自 ARM DynamIQ/DSU 与 x86 混合架构(E-core 簇共享 L2)。
- 现状映射:
  - arm 鲲鹏 920:`CONFIG_SCHED_CLUSTER=y` 已编译,`/proc/sys/kernel/sched_cluster` 已暴露 → 宿主调度域存在 CLUSTER 级。
  - x86 AMD EPYC 9T25:`CONFIG_SCHED_CLUSTER_PAIR=m`(非标准 CLUSTER),Zen 每核 L2 私有、仅 CCX/CCD 共享 L3,不存在「多核共享 L2」的 cluster → 等价决策在 MC/DIE(L3) 级。
- 不纳入原因:harness 是 4 vCPU、fork/exec 密集、唤醒/延迟敏感、单流为主,不是 L2 容量受限;4 vCPU guest 内不出现 CLUSTER 级(guest 侧无从谈起);「亲和 vs 摊薄」属二阶微调,收益被 ① NUMA 绑核完全盖住。
- 调度器实为 **EEVDF**(内核 6.6.0-72,CFS 已被替换,叠加 HCE/Euler 扩展)。经典 CFS 旋钮(`sched_latency_ns`/`sched_wakeup_granularity_ns`/`sched_migration_cost_ns`)**已不存在**(6.6 起被 EEVDF 删除)。实测可用旋钮:debugfs `base_slice_ns=2.8ms`、`migration_cost_ns=0.5ms`、`nr_migrate=32`;sysctl `sched_migration_cost`、`sched_protect_slice_timeout_ns`、`sched_prio_load_balance_enabled`、`sched_dynamic_affinity_disable`、`sched_util_low_pct`;arm 另有 `sched_cluster`(`CONFIG_SCHED_CLUSTER=y`)。对 4 vCPU 单流 harness 仍是二阶微调,优先级低。

### 11.【低】irqbalance 全核 IRQ 亲和(IRQ 未从 vCPU 核隔离)
- 含义:两节点 `irqbalance=active`,`/proc/irq/default_smp_affinity` = 全核,网卡/存储/timer IRQ 可在任意核投递,未被引导离开 vCPU 核。
- 作用:harness 短命令期间,宿主网卡(冷 npm)/存储/定时器 IRQ 随机抢占 vCPU 核 → 抖动。
- 优化方向:配合 isolcpus 把 IRQ 亲和从 vCPU 核挪走(`/proc/irq/<n>/smp_affinity_list`),或关 irqbalance 改静态绑定;低风险,可在线验证。

### 12.【已最优】无需动
- mitigations=off(x86)/ nospectre_bhb(arm)宿主缓解已关,与 guest 侧 mitigations=off 的 2.69× syscall 收益一致。
- cpufreq=performance、THP=always、swappiness=0、fs.file-max/nr_open/pid_max/threads-max 均已调顶。
- 沙箱存储加速层(diff-cow):CDC 分块(128KiB~1MiB)+ AES + redis 分层缓存(dragonfly 8GB)+ 本地 FS 回源(`/var/lib/sandbox-shared/manifest-storage`),配置已合理;热 npm 4318 文件写路径瓶颈更可能在内核虚拟化(EPT)而非存储层。

## 三、结论

宿主 CPU 富余、THP/mitigations 已到位;真正短板是「无 NUMA 绑核 + 无隔离核 + halt_poll 收缩关闭」。
性价比排序:①NUMA 绑核绑内存(消方差)> ②halt_poll_ns(提延迟,代价≈0)> ③tdp_mmu=Y(EPT 缺页)> ④isolcpus/nohz_full(去 jitter)> ⑤AVIC(收益可能大但兼容性风险高)。
cluster 调度域(CLS)已评估:arm 存在该级、x86 基本无;对 4 vCPU 单流 harness 属二阶微调,低杠杆,不纳入优化清单。