# ARM PVSPINLOCK（PV qspinlock）分析文档

> 分析对象：
> 1. 节点 77（10.30.42.77）`/home/gzk/kernel/` 下的内核源码（Huawei/openEuler 定制版 6.6.0）
> 2. openEuler 内核 PR #15513「Support pv-spinlock feature」（https://gitee.com/openeuler/kernel/pulls/15513）
>
> 日期：2026-11-17

---

## 1. 结论摘要

| 问题 | 结论 |
|---|---|
| 节点 77 现有内核（HZ=100 build）是否支持 arm PVSPINLOCK？ | **不支持**。它用的是 **CNA（NUMA-aware）qspinlock**，`CONFIG_PARAVIRT_SPINLOCKS` 在 arm64 上根本不存在。 |
| arm64 是否具备 PV qspinlock 的实现基础？ | **有**。通用 `kernel/locking/qspinlock_paravirt.h` + `qspinlock.c` 的 `_GEN_PV_LOCK_SLOWPATH` 分支已随 6.6 存在，只是被 `CONFIG_PARAVIRT_SPINLOCKS` 挡住、在 arm64 上没编译。 |
| PR #15513 做了什么？ | 补上 arm64 侧的「胶水」：新增 SMCCC 超调 `KICK_CPU`、guest 侧 static_call 接口、host（KVM）侧 `pv_unhalted` 唤醒 + `yield_to` boost。**这版未合入**（closed + ci_failed + needs-squash）。 |
| PV 与 CNA 关系 | **互斥**（都替换 slowpath）。要启用 PV 必须 `CONFIG_NUMA_AWARE_SPINLOCKS=n`。 |

---

## 2. 背景概念

### 2.1 为什么要 PV spinlock（核心痛点）

在 vCPU 超卖（overcommit）的虚拟化场景下：

1. vCPU A 持有自旋锁，但被宿主调度器抢占（preempted），物理 CPU 被分给别的 vCPU；
2. vCPU B/C 抢这把锁，只能空转自旋等 A；
3. A 不在跑，B/C 的自旋是**纯浪费物理 CPU**，还加剧调度颠簸。

x86 的 pvqspinlock 解法：**等待者自旋一个阈值后主动 HLT 睡下**，锁持有者释放时**踢（kick）醒**等待者；必要时把物理 CPU **让出（yield_to）给被抢占的锁持有者**，让它赶紧跑完释放锁。arm64 的 PV qspinlock 是同一思想，把 HLT 换成 **WFI**，把 x86 hypercall 换成 **SMCCC `KICK_CPU`**。

### 2.2 arm64 现状：为什么「原生」不支持

6.6 主线/OLK-6.6 里，PV qspinlock 的通用实现（`kernel/locking/qspinlock_paravirt.h`）早已存在，但它的编译开关 `CONFIG_PARAVIRT_SPINLOCKS` 只有 x86（`arch/x86/Kconfig:808`）和 powerpc pseries 定义。arm64 没有这个 Kconfig 项，因此：

- `kernel/locking/qspinlock.c:85`：`#if defined(CONFIG_PARAVIRT_SPINLOCKS) || defined(CONFIG_NUMA_AWARE_SPINLOCKS)` —— arm64 命中的是后者（CNA）。
- arm64 的 slowpath 走 `cna_queued_spin_lock_slowpath`，不是 `__pv_queued_spin_lock_slowpath`。

PR #15513 要做的正是「给 arm64 补上这个开关 + 架构胶水」。

---

## 3. 节点 77 现有内核源码排查结果（只读侦察）

### 3.1 `.config` 关键项

```
CONFIG_PARAVIRT=y
# CONFIG_PARAVIRT_SCHED is not set
# CONFIG_PARAVIRT_TIME_ACCOUNTING is not set
CONFIG_NUMA_AWARE_SPINLOCKS=y
CONFIG_ARCH_USE_QUEUED_SPINLOCKS=y
CONFIG_QUEUED_SPINLOCKS=y
CONFIG_QUEUED_RWLOCKS=y
（无 CONFIG_PARAVIRT_SPINLOCKS）
```

### 3.2 arm64 实际使用的 qspinlock 增强 = CNA

- `arch/arm64/include/asm/qspinlock.h`：slowpath → `cna_queued_spin_lock_slowpath`。
- `kernel/locking/qspinlock.c`（`_GEN_CNA_LOCK_SLOWPATH` 分支）把 `pv_wait_head_or_lock`、`mcs_lock_handoff` 等 hook 重定义为 `cna_*`，生成 `__cna_queued_spin_lock_slowpath`。
- 作用：减少跨 NUMA 节点 cacheline 弹跳（节点 77 是 2×80 核双 NUMA 节点）。**它不感知 vCPU 抢占**，与 PV 是两码事。

### 3.3 arm64 半虚拟化只有两类（Huawei 用 SMCCC 实现，均非 pvspinlock）

| 机制 | 超调 | 作用 |
|---|---|---|
| PV time（steal time） | `0xC5000020/0x21`（标准 ARM DEN0057A） | vCPU 被抢占时间统计 |
| PV sched（vCPU preempted 检测） | `0xC5000090/0x91/0x92`（Huawei 私有） | `vcpu_is_preempted()` → 锁乐观自旋时提前让出 |

- `arch/arm64/kernel/paravirt-spinlocks.c`（18 行）：只注册 `__native_vcpu_is_preempted` 的 static_call 桩，名字带 "spinlocks" 但**不是 pvqspinlock**。
- `vcpu_is_preempted()` 消费点：`kernel/locking/osq_lock.c:146`（前驱 vCPU 被抢占则 break）、`kernel/locking/mutex.c:371`、`kernel/sched/core.c:7411`。
- 当前 build 里 `CONFIG_PARAVIRT_SCHED` 未开，故 `arch/arm64/include/asm/spinlock.h:23` 走 `#else` 分支，`vcpu_is_preempted()` 恒返回 `false` —— **这套「类似 pvspinlock 的收益」也没生效**。

---

## 4. PR #15513 详细分析

### 4.1 PR 概况

- 标题：`Support pv-spinlock feature`；5 个 commit；作者 `Zengruan Ye <yezengruan@huawei.com>`（`paravirt.c`/`pvsched` 同作者）。
- 目标分支：`OLK-6.6`；18 个文件。
- 5 个 commit：① 加 SMCCC kick 超调 → ② host 实现 `KICK_CPU` → ③ guest 加 PV qspinlock 接口 → ④ 启用 PV qspinlock → ⑤ 加 tracepoint。
- **状态：已关闭、无法自动合并、`ci_failed` / `stat/needs-squash`** → 未合入。

### 4.2 设计思路（三条主线）

1. **自旋 → WFI 睡 → 被踢醒**：把「抢不到锁就一直自旋」改成「自旋一个阈值后 WFI 睡下」，锁释放时 `KICK_CPU` 踢醒。
2. **static_call 运行时打补丁**：默认 native（dummy），探测到宿主支持 + 开 `arm_pvspin` 才热替换为 PV 版本，native 内核零开销。
3. **host 侧 `pv_unhalted` + `kvm_vcpu_yield_to`**：vCPU 因 WFI 阻塞时 `pv_unhalted=false`，被踢时 `=true` 使其重新 runnable；kick 时 `yield_to` 让出物理 CPU 给锁持有者（boost）。

### 4.3 完整运行流程（逐帧拆解）

**一句话剧情**：三个 vCPU 抢同一把锁。抢不到的**先原地转一小会儿**，转不到就**执行 `wfi` 躺下睡觉**（把物理 CPU 还给宿主），锁持有者释放时**通过 SMCCC 超调 `KICK_CPU` 把它喊醒**；必要时宿主还会**把自己正占的 CPU 让给被喊醒的 vCPU**。

下面用「vCPU0 持锁、vCPU1 抢锁、vCPU2 排队」这个具体场景，一帧一帧走一遍。

#### 阶段 0 · 先看懂「锁」和「排队」

自旋锁不是一个笼统的"转圈"，而是一根**队列**。

- 锁本体是 32 位字（`struct qspinlock { atomic_t val; }`），三个字段：
  - `bit0  locked` —— 锁在谁手里（1 = 有人持有）
  - `bit8  pending` —— "第二个人"的插队位；**PV 模式下被借用作 `_Q_SLOW_VAL`（"我要睡了"的标记）**
  - `bit16+ tail` —— 队尾节点编号

- 抢不到锁的人会排成一根 **MCS 队列**，每个等待者占一个 per-CPU 节点 `qnode`，节点里记着：`next`（排我后面的）、`state`（我是醒着/睡着）、`cpu`（我在哪颗物理 CPU）。

```
锁(32位):  [ tail(16) | pending/慢(8) | locked(0) ]
                 │ 指向队尾
队列:   vCPU0(持有锁) ← vCPU1(队头) ← vCPU2(队尾)
             每个节点 = { next, state, cpu }
```

排队规则（关键）：
- **队头**（vCPU1）盯着**锁本身**转；
- **队尾/中间**（vCPU2）盯着**前一个人的节点**转；
- 解锁时，锁从 vCPU0 交给**队头 vCPU1**，vCPU1 再踢给下一个。

还有一个 PV 特有的配角：因为睡着的人**不会主动"看到"锁被释放**，解锁者必须**主动去喊**，而"喊"要知道对方在哪个 CPU，所以 PV 维护一张 **per-lock 哈希表 `pv_lock_hash`**，记录 `(锁, 等待节点) → 等待者 CPU`。`pv_qspinlock_init()` 里的 `__pv_init_lock_hash()` 就是建这张表。

#### 阶段 1 · vCPU1 抢锁失败，进慢路径（guest 加锁）

vCPU1 调 `spin_lock()`。fast path 先 `cmpxchg` 抢 `locked` 位，失败（因为 vCPU0 拿着），于是进 `queued_spin_lock_slowpath()`：

```c
// arch/arm64/include/asm/qspinlock.h（开了 PV 后）
static inline void queued_spin_lock_slowpath(lock, val) {
    pv_queued_spin_lock_slowpath(lock, val);   // static_call 一跳
}
```

`pv_queued_spin_lock_slowpath` 默认是 native，但 `pv_qspinlock_init()` 已把它换成通用的 `__pv_queued_spin_lock_slowpath`。vCPU1 排进队列，成为**队头**，开始盯着 `lock->locked` 转圈。

#### 阶段 2 · 转满 512 次还抢不到 → WFI 睡觉（guest 等待）

队头 vCPU1 边转边数，转满 `_Q_PENDING_LOOPS = 1<<9 = 512` 次仍拿不到，就放弃傻转，改调：

```c
pv_wait(&lock->locked, _Q_SLOW_VAL);   // "锁还是老样子的话，我就睡"
```

这一句就是整套机制"睡下去"的触发点。arm64 里 `pv_wait` → static_call `pv_qspinlock_wait` → `kvm_wait`：

```c
static void kvm_wait(u8 *ptr, u8 val) {
    if (in_nmi()) return;
    local_irq_save(flags);
    if (READ_ONCE(*ptr) != val) goto out;   // ① 睡前再瞄一眼，锁变了就不睡
    dsb(sy);                                 // ② 屏障：保证上面那一眼先完成
    wfi();                                   // ③ 真正躺下
out: local_irq_restore(flags);
}
```

三个细节，回答了"为什么不会漏醒"：

1. **`READ_ONCE(*ptr) != val` 二次确认**：睡前最后瞄一眼，万一锁其实已经释放了，就直接 `goto out` 不睡，回去继续抢。
2. **`dsb(sy)` 内存屏障**：保证"瞄一眼"这次读先于 WFI 完成、让 CPU 看到最新值。否则可能出现：读到旧值 → 决定睡 → 但锁刚好在此时被释放并踢了我 → 我的 WFI 没赶上这次唤醒 → 永久睡死。
3. **`wfi()`**：真正进入低功耗等待。**宿主 KVM 会捕获这条 WFI，把 vCPU1 调度出去**，物理 CPU 让给别人——这就是"不再空转烧 CPU"的核心。

#### 阶段 3 · vCPU2 排在队尾，睡在前驱上（guest 等待）

vCPU2 也来抢，发现锁已被 vCPU1 占了、vCPU1 又在队列里，于是 vCPU2 排到**队尾**。队尾的人**不看锁本身**，而是看**前一个节点**：

```c
// 通用 PV 慢路径里，中间/队尾等待者:
pv_wait(&prev->state, vcpu_halted);   // 盯着前驱的状态睡，前驱走时会被踢醒
```

所以 vCPU2 同样 WFI 睡下，只是它等的是"前一个人 vCPU1"，不是锁。

#### 阶段 4 · 宿主捕获 WFI，vCPU1/vCPU2 被阻塞（host）

guest 执行 `wfi` 会 trap 到 EL2，KVM 的 `kvm_handle_wfx()` 接住：

```c
// arch/arm64/kvm/handle_exit.c
vcpu->stat.wfi_exit_stat++;
vcpu->arch.pvsched.pv_unhalted = false;   // ★ 记下：这 vCPU 睡了
```

vCPU1、vCPU2 从此进入"不可运行"状态，宿主调度器把物理 CPU 分给别的 vCPU。

#### 阶段 5 · vCPU0 走完临界区，解锁踢醒队头（guest 解锁）

假设 vCPU0 恢复运行、走完临界区，调 `spin_unlock()`。PV 模式下 `queued_spin_unlock()` → `pv_queued_spin_unlock()` → 通用 `__pv_queued_spin_unlock`。它做三件事：

1. 从队列里取出下一个等待者（vCPU1 的节点）；
2. 从 `pv_lock_hash` 查到 vCPU1 睡在哪个物理 CPU；
3. 调 `pv_kick(cpu)` 去喊它：

```c
// guest: pv_kick → static_call pv_qspinlock_kick → kvm_kick_cpu
static void kvm_kick_cpu(int cpu) {
    arm_smccc_1_1_invoke(ARM_SMCCC_HV_PV_SCHED_KICK_CPU, cpu, &res);  // 发超调
    trace_kvm_kick_cpu(...);
}
```

→ 执行 `hvc #0`，function ID `0xC5000093` 放 x0、目标 CPU 放 x1，陷入 EL2。

#### 阶段 6 · 宿主接住 KICK_CPU，唤醒 + 让出（host kick）

KVM 的 `kvm_smccc_call_handler()` 按 function ID 分发到：

```c
case ARM_SMCCC_HV_PV_SCHED_KICK_CPU:
    val[0] = kvm_pvsched_kick_vcpu(vcpu);
```

`kvm_pvsched_kick_vcpu()` 干四件事：

```c
target = kvm_get_vcpu(kvm, smccc_get_arg1(vcpu));  // x1 = 目标 vCPU = vCPU1
target->arch.pvsched.pv_unhalted = true;           // ① 标记"已被踢醒"
kvm_make_request(KVM_REQ_IRQ_PENDING, target);     // ② 置请求位
kvm_vcpu_kick(target);                             // ③ 把目标从阻塞态拉起来
if (READ_ONCE(target->ready))
    kvm_vcpu_yield_to(target);                     // ④ 关键：把 CPU 让给目标
```

- **①②③** 的效果：`kvm_arch_vcpu_runnable()` 因为 `pv_unhalted == true` 对 vCPU1 返回 true，于是 vCPU1 重新变得"可调度"。
- **④ `kvm_vcpu_yield_to(target)`**：如果 vCPU0 现在占的这颗物理 CPU 恰好是 vCPU1 想跑的那颗，vCPU0 就**把 CPU 让给 vCPU1**。这就是"boost"——**当锁持有者被抢占时，让出能让它立刻回到 CPU 上、赶紧跑完释放锁**。这是 PV spinlock 在 vCPU 超卖场景下最大的收益来源。

#### 阶段 7 · vCPU1 醒来拿锁；之后它解锁再踢 vCPU2（循环）

vCPU1 被唤醒后，`kvm_wait` 里的 `wfi()` 返回，`__pv_queued_spin_lock_slowpath` 继续抢锁 → 拿到锁 → 进临界区 → 走完 → 解锁时**用同样的 `pv_kick` 去踢醒队尾的 vCPU2**。如此循环。

#### 一张总结表

| # | 动作 | 谁执行 | 触发点 | 关键代码 |
|---|---|---|---|---|
| 1 | 抢锁失败进慢路径 | guest vCPU1 | fast path cmpxchg 失败 | `queued_spin_lock_slowpath()` → `pv_queued_spin_lock_slowpath()` |
| 2 | 队头自旋 512 次后睡 | guest vCPU1 | 转满 `_Q_PENDING_LOOPS` | `pv_wait(&lock->locked, _Q_SLOW_VAL)` → `kvm_wait()` → `wfi()` |
| 3 | 队尾睡在前驱上 | guest vCPU2 | 排队尾 | `pv_wait(&prev->state, vcpu_halted)` |
| 4 | 捕获 WFI 阻塞 vCPU | host KVM | WFI trap | `kvm_handle_wfx()`：`pv_unhalted=false` |
| 5 | 解锁、踢醒队头 | guest vCPU0 | `spin_unlock()` | `__pv_queued_spin_unlock()` → `pv_kick()` → `kvm_kick_cpu()` |
| 6 | 发 KICK_CPU 超调 | guest vCPU0 | kick | `arm_smccc_1_1_invoke(0xC5000093, cpu)` |
| 7 | 唤醒 + 让出 | host KVM | 超调分发 | `kvm_pvsched_kick_vcpu()`：`pv_unhalted=true` + `vcpu_kick` + `yield_to` |
| 8 | 醒来继续抢锁 | guest vCPU1 | 被踢醒 | `__pv_queued_spin_lock_slowpath()` 继续 |

### 4.4 逐文件代码分析

#### ① 新超调 ID —— `include/linux/arm-smccc.h`
```c
#define ARM_SMCCC_HV_PV_SCHED_KICK_CPU \
    ARM_SMCCC_CALL_VAL(FAST_CALL, SMC_64, OWNER_STANDARD_HYP, 0x93)   // = 0xC5000093
```
与 pvsched 族（`0x90/0x91/0x92`）同族，Huawei 私有、非 ARM 标准；文档 `pvsched.rst` 对应 `0xC5000093`。

#### ② 配置开关 —— `arch/arm64/Kconfig`
```c
config PARAVIRT_SPINLOCKS
    bool "Paravirtualization layer for spinlocks"
    depends on PARAVIRT && SMP && !NUMA_AWARE_SPINLOCKS   // ★ 与 CNA 互斥
```
`openeuler_defconfig` 写死 `CONFIG_PARAVIRT_SPINLOCKS=n`（默认关）。

#### ③ guest 侧静态调用接口 —— `arch/arm64/include/asm/paravirt.h`
声明 4 个 static_call 钩子，默认全 dummy（native），并包装成通用名 `pv_wait()`/`pv_kick()` 供通用 `qspinlock_paravirt.h` 调用：
```c
DECLARE_STATIC_CALL(pv_qspinlock_queued_spin_lock_slowpath, dummy_...);
DECLARE_STATIC_CALL(pv_qspinlock_queued_spin_unlock,         dummy_...);
DECLARE_STATIC_CALL(pv_qspinlock_wait,                       dummy_wait);
DECLARE_STATIC_CALL(pv_qspinlock_kick,                       dummy_kick);
```

#### ④ guest 侧 slowpath 分派 —— `arch/arm64/include/asm/qspinlock.h`
由单一 CNA 分支改为三分支：
```c
#ifdef CONFIG_NUMA_AWARE_SPINLOCKS
    ... CNA 分支（原有）
#else
  #if defined(CONFIG_SMP) && defined(CONFIG_PARAVIRT_SPINLOCKS)
    #define _Q_PENDING_LOOPS (1 << 9)   // pending 位/队头自旋阈值（x86 同款）
    queued_spin_lock_slowpath() { pv_queued_spin_lock_slowpath(lock, val); }
    queued_spin_unlock()        { pv_queued_spin_unlock(lock); }
  #endif
#endif
#include <asm-generic/qspinlock.h>
```

#### ⑤ guest 侧 PV 实现 —— `arch/arm64/kernel/paravirt.c`（核心）
```c
static void kvm_kick_cpu(int cpu) {
    arm_smccc_1_1_invoke(ARM_SMCCC_HV_PV_SCHED_KICK_CPU, cpu, &res);  // 踢醒
    trace_kvm_kick_cpu(...);
}
static void kvm_wait(u8 *ptr, u8 val) {
    if (in_nmi()) return;
    local_irq_save(flags);
    if (READ_ONCE(*ptr) != val) goto out;   // 二次确认，防丢唤醒
    dsb(sy);                                // ★ 内存屏障，保证读 *ptr 先于 WFI
    wfi();                                  // ★ 睡眠
    trace_kvm_wait(...);
out: local_irq_restore(flags);
}
DEFINE_STATIC_CALL(pv_qspinlock_queued_spin_lock_slowpath, native_queued_spin_lock_slowpath);
DEFINE_STATIC_CALL(pv_qspinlock_queued_spin_unlock,         native_queued_spin_unlock);
DEFINE_STATIC_CALL(pv_qspinlock_wait, kvm_wait);
DEFINE_STATIC_CALL(pv_qspinlock_kick, kvm_kick_cpu);

void __init pv_qspinlock_init(void) {
    if (num_possible_cpus() == 1) arm_pvspin = false;
    if (!arm_pvspin) { pr_info("PV qspinlocks disabled\n"); return; }
    __pv_init_lock_hash();   // 分配 PV lock hash 表（通用 qspinlock_paravirt.h 提供）
    static_call_update(pv_qspinlock_queued_spin_lock_slowpath, __pv_queued_spin_lock_slowpath);
    static_call_update(pv_qspinlock_queued_spin_unlock,         __pv_queued_spin_unlock);
    static_call_update(pv_qspinlock_wait, kvm_wait);
    static_call_update(pv_qspinlock_kick, kvm_kick_cpu);
}
early_param("arm_pvspin", arm_parse_pvspin);   // 默认 false，需启动参数 arm_pvspin 开启
```

**要点**：
- `dsb(sy)` + `wfi()` 是 ARM 经典丢唤醒防护：先屏障保证 `READ_ONCE(*ptr)` 结果可见，再 WFI，避免「检查到没变化 → 正要睡 → 锁恰好释放并 kick → WFI 错过 kick」竞态。
- `__pv_queued_spin_lock_slowpath` / `__pv_queued_spin_unlock` / `__pv_init_lock_hash` 均来自**已存在的通用** `kernel/locking/qspinlock_paravirt.h` + `qspinlock.c`（`_GEN_PV_LOCK_SLOWPATH` 分支），非本 PR 新增。

#### ⑥ guest 侧声明头 —— `arch/arm64/include/asm/qspinlock_paravirt.h`（新增，12 行）
仅 `extern void __pv_queued_spin_unlock(struct qspinlock *lock);`。

#### ⑦ host 侧（KVM）核心 —— `arch/arm64/kvm/pvsched.c`
```c
long kvm_pvsched_kick_vcpu(struct kvm_vcpu *vcpu) {
    vcpu_idx = smccc_get_arg1(vcpu);            // x1 = 目标 vCPU id
    target = kvm_get_vcpu(kvm, vcpu_idx);
    if (!target) return SMCCC_RET_NOT_SUPPORTED;
    target->arch.pvsched.pv_unhalted = true;    // ★ 标记「已被踢醒」
    kvm_make_request(KVM_REQ_IRQ_PENDING, target);
    kvm_vcpu_kick(target);                      // ★ 唤醒
    if (READ_ONCE(target->ready))
        kvm_vcpu_yield_to(target);              // ★ 让出物理 CPU（boost）
    trace_kvm_pvsched_kick_vcpu(vcpu->vcpu_id, target->vcpu_id);
    return SMCCC_RET_SUCCESS;
}
```
`kvm_hypercall_pvsched_features()` 把 `KICK_CPU` 加入已支持特性列表。

#### ⑧ host 侧可运行判定 —— `arch/arm64/kvm/arm.c`
```c
int kvm_arch_vcpu_runnable(struct kvm_vcpu *v) {
    bool pv_unhalted = v->arch.pvsched.pv_unhalted;
    return ((irq_lines || kvm_vgic_vcpu_pending_irq(v) || pv_unhalted)
            && !stopped && !pause);
}
```
闭环关键：vCPU 因 WFI 睡了 → 被 kick 后 `pv_unhalted=true` → 重新 runnable。

#### ⑨ host 侧 WFI 处理 —— `arch/arm64/kvm/handle_exit.c`
```c
// kvm_handle_wfx() 的 WFI 分支
vcpu->stat.wfi_exit_stat++;
vcpu->arch.pvsched.pv_unhalted = false;   // ★ WFI 时置 false（睡下去）
```

#### ⑩ host 侧超调分发 —— `arch/arm64/kvm/hypercalls.c`
```c
case ARM_SMCCC_HV_PV_SCHED_KICK_CPU:
    val[0] = kvm_pvsched_kick_vcpu(vcpu);
    break;
```

#### ⑪ 数据结构 —— `arch/arm64/include/asm/kvm_host.h`
```c
#ifdef CONFIG_PARAVIRT_SCHED
    struct { bool pv_unhalted; gpa_t base; } pvsched;   // 新增 pv_unhalted
#endif
```

#### ⑫ 文档 & trace
- `Documentation/virt/kvm/arm/pvsched.rst`：描述 `KICK_CPU 0xC5000093` 与 WFI 等待/唤醒语义。
- `arch/arm64/kernel/trace-paravirt.h`（新增）：guest 侧 `kvm_kick_cpu`/`kvm_wait` tracepoint。
- `arch/arm64/kvm/trace_arm.h`：host 侧 `kvm_pvsched_kick_vcpu` tracepoint。

---

## 5. 对比总表

| 维度 | 节点 77 现有内核（HZ=100 build） | PR #15513（若合入并启用） |
|---|---|---|
| spinlock 慢路径 | CNA（`CONFIG_NUMA_AWARE_SPINLOCKS=y`） | PV qspinlock |
| `CONFIG_PARAVIRT_SPINLOCKS` | 不存在 | 新增，`depends on !NUMA_AWARE_SPINLOCKS` |
| 睡眠/唤醒原语 | 无（纯自旋，仅 CNA 重排） | WFI / SMCCC `KICK_CPU` |
| 对 vCPU 抢占感知 | 无 | 有（`yield_to` boost + `preempted`） |
| 能否并存 | —— | 互斥 |

---

## 6. 启用前置条件（要真正跑通 PV qspinlock）

1. **重编内核**：`CONFIG_NUMA_AWARE_SPINLOCKS=n` + `CONFIG_PARAVIRT_SPINLOCKS=y`（建议同时 `CONFIG_PARAVIRT_SCHED=y`）。
2. **宿主 KVM 合入 host 侧改动**：`pv_unhalted`、`kvm_pvsched_kick_vcpu()`、`kvm_arch_vcpu_runnable()` 的 `pv_unhalted` 判定、`handle_exit.c` 的 `pv_unhalted=false`；否则 `KICK_CPU` 超调无人处理。
3. **guest 启动参数**：加 `arm_pvspin`（默认 false，否则 `PV qspinlocks disabled`）。
4. **宿主被探测到**：`has_kvm_pvsched()` 返回成功（宿主实现 pvsched 含 KICK_CPU）。

---

## 7. 关键坑 / 注意点

1. **runtime 默认关**：`arm_pvspin` 默认 `false`，不加启动参数不会启用。
2. **隐式依赖 `PARAVIRT_SCHED`**：`pv_qspinlock_init()` 唯一调用点在 `pv_sched_init()`（`#ifdef CONFIG_PARAVIRT_SCHED` 内），而 Kconfig 里 `PARAVIRT_SPINLOCKS` **没有 `select PARAVIRT_SCHED`**。只开 `PARAVIRT_SPINLOCKS` 不开 `PARAVIRT_SCHED` → 特性**静默失效**（slowpath 永远 native）。
3. **与 CNA 互斥**：`depends on !NUMA_AWARE_SPINLOCKS`，两者只能选一。
4. **宿主必须配套**：guest 侧改动无宿主支持则无效。
5. **两个独立机制别混淆**：
   - `preempted`（pvsched）：宿主调度出去时置 1，喂 `vcpu_is_preempted()`（osq_lock/mutex 用）。
   - `pv_unhalted`（本 PR 新增）：WFI 睡下 / 被 kick 醒的状态，喂 `kvm_arch_vcpu_runnable()`。
6. **本 PR 未合入**：`closed` + `ci_failed` + `needs-squash`，落地需 squash 重提过 CI。

---

## 8. 附：相关 SMCCC 超调一览

| 名称 | ID | 标准/私有 | 用途 |
|---|---|---|---|
| `PV_TIME_FEATURES` | `0xC5000020` | 标准（ARM DEN0057A） | 探测 PV time |
| `PV_TIME_ST` | `0xC5000021` | 标准（ARM DEN0057A） | 读取 stolen time |
| `PV_SCHED_FEATURES` | `0xC5000090` | Huawei 私有 | 探测 pvsched |
| `PV_SCHED_IPA_INIT` | `0xC5000091` | Huawei 私有 | 传递 vCPU state 的 IPA |
| `PV_SCHED_IPA_RELEASE` | `0xC5000092` | Huawei 私有 | 释放 vCPU state IPA |
| `PV_SCHED_KICK_CPU` | `0xC5000093` | Huawei 私有（PR #15513 新增） | 踢醒 WFI 睡眠的 vCPU |
