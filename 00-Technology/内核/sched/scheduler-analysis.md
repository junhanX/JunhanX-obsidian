# Linux Kernel 6.18.34 调度器 —— 从架构到代码的完整分析

> 分析日期: 2026-06-02
> 内核版本: Linux 6.18.34
> 源码路径: `kernel/sched/` (~55,844 行, ~40 个文件)
> 分析方法: 源码逐段阅读

---

## 目录

1. [整体架构](#1-整体架构)
2. [调度类机制](#2-调度类机制)
3. [调度流程全景](#3-调度流程全景)
4. [每 CPU 运行队列](#4-每-cpu-运行队列)
5. [EEVDF 算法 —— 代码级分析](#5-eevdf-算法代码级分析)
6. [唤醒抢占](#6-唤醒抢占)
7. [Tick 驱动路径](#7-tick-驱动路径)
8. [Deadline 调度](#8-deadline-调度)
9. [DL Server 状态机](#9-dl-server-状态机)
10. [实时调度](#10-实时调度)
11. [SCHED_PROXY_EXEC 代理执行](#11-sched_proxy_exec-代理执行)
12. [负载均衡](#12-负载均衡)
13. [PELT 负载跟踪](#13-pelt-负载跟踪)
14. [唤醒路径](#14-唤醒路径)
15. [调度器特性开关](#15-调度器特性开关)
16. [关键数据结构关系](#16-关键数据结构关系)
17. [本版本架构创新](#17-本版本架构创新)

---

## 1. 整体架构

调度器位于 `kernel/sched/`，采用**可插拔调度类**设计。各类通过链接器脚本段按优先级排列：

```
优先级高 → 低
┌──────────────────────────────────────────┐
│  stop_task  │ 停机任务（热迁移、热补丁）   │ stop_task.c  117 行
├──────────────────────────────────────────┤
│  deadline   │ SCHED_DEADLINE（EDF+CBS）  │ deadline.c   3,739 行
├──────────────────────────────────────────┤
│  rt         │ SCHED_FIFO / SCHED_RR      │ rt.c         2,942 行
├──────────────────────────────────────────┤
│  ext        │ sched_ext（BPF 可扩展）    │ ext.c        6,987 行
├──────────────────────────────────────────┤
│  fair       │ CFS/EEVDF（普通任务）      │ fair.c       14,089 行
├──────────────────────────────────────────┤
│  idle       │ 空闲任务                   │ idle.c       568 行
└──────────────────────────────────────────┘
```

### 关键文件

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `core.c` | ~10,901 | 调度入口、上下文切换、唤醒、代理执行、tick |
| `fair.c` | ~14,089 | EEVDF 算法、CFS 队列、负载均衡、带宽控制 |
| `deadline.c` | ~3,739 | EDF + CBS、deadline 队列、DL Server |
| `rt.c` | ~2,942 | FIFO/RR、RT 带宽、push/pull 迁移 |
| `sched.h` | ~3,929 | 内部数据结构定义 |
| `ext.c` | ~6,987 | sched_ext BPF 调度扩展 |
| `topology.c` | ~2,876 | CPU 拓扑、调度域 |
| `pelt.c` | — | PELT 负载跟踪 |
| `features.h` | 128 | 运行时调度特性开关 |
| `build_policy.c` | — | 策略构建 |
| `cpufreq_schedutil.c` | — | CPU 频率调节 |
| `psi.c` | — | 压力停滞指数 |

---

## 2. 调度类机制

### 2.1 struct sched_class (sched.h:2405)

```c
struct sched_class {
    // 任务入队/出队
    void (*enqueue_task)(struct rq *rq, struct task_struct *p, int flags);
    bool (*dequeue_task)(struct rq *rq, struct task_struct *p, int flags);
    void (*yield_task)(struct rq *rq);

    // 唤醒抢占
    void (*wakeup_preempt)(struct rq *rq, struct task_struct *p, int flags);

    // 负载均衡
    int (*balance)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);

    // 选择下一个任务
    struct task_struct *(*pick_task)(struct rq *rq);
    struct task_struct *(*pick_next_task)(struct rq *rq, struct task_struct *prev);

    // 任务切换前后
    void (*put_prev_task)(struct rq *rq, struct task_struct *p, struct task_struct *next);
    void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);

    // tick 处理
    void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);

    // 运行时更新
    void (*update_curr)(struct rq *rq);

    // ... 更多回调
};
```

### 2.2 优先级排列 (sched.h:2555)

调度类实例通过 GCC section 属性排列在内存中，高优先级类在**低地址**：

```c
#define DEFINE_SCHED_CLASS(name) \
    const struct sched_class name ## _sched_class \
    __section("__" #name "_sched_class") __used

// 链接器段顺序: stop → dl → rt → ext → fair → idle
// 内存地址: 低 ←─────────────────────→ 高
```

因此比较优先级只需要比较地址：

```c
#define sched_class_above(_a, _b)  (_a < _b)   // 地址低 = 优先级高
```

### 2.3 遍历宏

```c
// 遍历所有调度类（stop → dl → rt → ext → fair → idle）
for_each_class(class)
    for (class = __sched_class_highest; class; class = class->next)

// 只遍历活跃的调度类（跳过未启用的 ext）
for_each_active_class(class)
```

---

## 3. 调度流程全景

### 3.1 进入调度的三条路径

```
1. 显式阻塞 (voluntary sleep)
   mutex_lock() / semaphore_down() / wait_event()
   → schedule()

2. TIF_NEED_RESCHED 标志 (preemption)
   tick 中断中 sched_tick() 设置标志
   → 中断返回路径检查标志 → schedule()

3. 唤醒 (wakeup)
   wake_up() / ttwu() 设置 TIF_NEED_RESCHED
   → 最近的可抢占点 → schedule()
```

### 3.2 schedule() 完整链路 (core.c:7022)

```c
asmlinkage __visible void __sched schedule(void)
{
    sched_submit_work(tsk);         // ① 调度前: 刷新 workqueue/plug
    __schedule_loop(SM_NONE);       // ② 调度循环
    sched_update_worker(tsk);       // ③ 调度后: 处理 worker
}
```

### 3.3 __schedule_loop() (core.c:7013)

```c
static __always_inline void __schedule_loop(int sched_mode)
{
    do {
        preempt_disable();                      // 禁止抢占
        __schedule(sched_mode);                 // 核心调度
        sched_preempt_enable_no_resched();      // 重新允许抢占（不触发 resched）
    } while (need_resched());                   // 如果还有 need_resched → 继续
}
```

**为什么需要循环？** 因为 `__schedule()` 选出的下一个任务可能又设置了 `need_resched`（例如被另一个更高优先级任务唤醒），所以要循环直到确定没有 pending 的调度请求。

### 3.4 `__schedule()` 核心 (core.c:6790)

```c
static void __schedule(int sched_mode)
{
    struct task_struct *prev, *next;
    struct rq *rq;
    int cpu;

    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    prev = rq->curr;

    schedule_debug(prev);                       // 调试检查

    // ① 更新时钟
    update_rq_clock(rq);

    // ② 如果 prev 状态改变了（自愿休眠），移出队列
    if (prev->state != TASK_RUNNING)
        try_to_block_task(rq, prev, ...);

    // ③ 选择下一个任务（donor 是调度上下文，可能 ≠ curr）
    next = pick_next_task(rq, rq->donor, &rf);

    // ④ Proxy exec: 如果选出的任务被互斥锁阻塞
    if (unlikely(task_is_blocked(next))) {
        // 跟随 blocked_on 链，找到可运行的锁持有者
        next = find_proxy_task(rq, next, &rf);
        // 如果找不到 → 进入 idle
    }

    // ⑤ 上下文切换
    context_switch(rq, prev, next, &rf);
}
```

### 3.5 pick_next_task() (core.c:5956)

```c
static inline struct task_struct *
__pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    rq->dl_server = NULL;

    /* ── 快速路径 ──────────────────────────────────────
     * 如果 prev 的调度类 ≤ fair（没有 RT/Deadline 任务），
     * 且所有可运行任务都属于 CFS 类，直接走 CFS 路径。
     * 这是绝大多数服务器和桌面场景的命中路径。
     */
    if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
               rq->nr_running == rq->cfs.h_nr_queued)) {

        p = pick_next_task_fair(rq, prev, rf);
        if (!p) {
            p = pick_task_idle(rq);              // fallback 到 idle
            put_prev_set_next_task(rq, prev, p);
        }
        return p;
    }

    /* ── 慢速路径 ─────────────────────────────────────
     * 存在非 CFS 任务时，从高到低遍历所有调度类。
     */
    prev_balance(rq, prev, rf);  // 从 prev 的类向下执行 balance

    for_each_active_class(class) {
        if (class->pick_next_task) {
            p = class->pick_next_task(rq, prev);
            if (p) return p;
        } else {
            p = class->pick_task(rq);
            if (p) {
                put_prev_set_next_task(rq, prev, p);
                return p;
            }
        }
    }
}
```

### 3.6 上下文切换 context_switch() (core.c:5275)

```c
context_switch(rq, prev, next, &rf)
{
    struct mm_struct *mm = next->mm;
    struct mm_struct *oldmm = prev->active_mm;

    /* ── 地址空间切换 ── */
    if (!mm) {
        // 内核线程 → 进入 lazy TLB 模式
        next->active_mm = oldmm;
        enter_lazy_tlb(oldmm, next);
    } else {
        // 用户态任务 → 切换页表
        switch_mm_irqs_off(oldmm, mm, next);
    }

    switch_mm_cid(prev, next);        // 并发 ID 切换
    prepare_lock_switch(rq, next);    // 锁准备

    /* ── 寄存器/栈切换 (架构相关汇编) ── */
    switch_to(prev, next, prev);

    finish_task_switch(prev);         // 清理前一个任务
}
```

---

## 4. 每 CPU 运行队列

### 4.1 声明 (core.c:126)

```c
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);
```

`__percpu` + cache line 对齐，避免 false sharing。

### 4.2 struct rq (sched.h:1120)

```c
struct rq {
    raw_spinlock_t        __lock;            // rq 锁，保护以下所有字段
    unsigned int          nr_running;        // 可运行任务总数
    unsigned long         nr_switches;       // 上下文切换计数

    /* 三个子运行队列 */
    struct cfs_rq         cfs;               // CFS/EEVDF 队列
    struct rt_rq          rt;                // RT 队列
    struct dl_rq          dl;                // Deadline 队列
    struct scx_rq         scx;               // sched_ext 队列（可选）

    /* DL Server (CFS 带宽保证) */
    struct sched_dl_entity fair_server;

    /* 双上下文 (SCHED_PROXY_EXEC) */
    struct task_struct __rcu *donor;         // 调度上下文（被挑选的任务）
    struct task_struct __rcu *curr;          // 执行上下文（真正在 CPU 上跑的）

    struct task_struct    *idle;             // per-CPU 空闲任务
    struct task_struct    *stop;             // per-CPU 停机任务

    /* 时钟 */
    u64                   clock;             // 运行队列时钟
    u64                   clock_task;        // 任务时钟（排除 IRQ/steal 等）
    u64                   clock_pelt;        // PELT 负载跟踪时钟

    /* 负载均衡 */
    struct root_domain    *rd;               // 根调度域
    struct sched_domain __rcu *sd;           // 调度域树
    unsigned long         cpu_capacity;      // CPU 容量

    /* 任务列表 */
    struct list_head      cfs_tasks;         // 本 CPU 上所有 CFS 任务列表
    // ...
};
```

**关键概念**: `donor` 和 `curr` 的分离是 SCHED_PROXY_EXEC 的基础。正常情况下 `donor == curr`；当代理执行时，`donor` 是被阻塞的高优先级任务（其调度属性决定调度决策），`curr` 是实际在 CPU 上运行的锁持有者。

---

## 5. EEVDF 算法 —— 代码级分析

这是本版本最大的变革。内核已从传统 CFS 的"最小 vruntime 优先"转向 **EEVDF（Earliest Eligible Virtual Deadline First）**。

### 5.1 核心概念

EEVDF 的选择基于两个条件：
1. **合格性 (Eligibility)**: 任务是否"欠服务"（得到的服务 < 应得的公平份额）
2. **紧急度 (Urgency)**: 在合格任务中，谁的虚拟截止时间最早

### 5.2 数学公式

```
虚拟截止时间:  vd = ve + r/w

其中:
  ve = vruntime（虚拟运行时间，按权重缩放）
  r  = 请求的服务量（slice，默认 sysctl_sched_base_slice）
  w  = 实体权重（由 nice 值决定）

Lag (lag 表示欠/超服务量):
  lag_i = S - s_i = w_i × (V - v_i)

其中:
  S = 加权总服务量
  s_i = 实体 i 的加权服务量
  V = 加权平均 vruntime（理想服务点）

合格条件:
  lag >= 0  ⟺  V >= v_i
  即：实体的 vruntime 不超过加权平均值，说明它欠服务
```

### 5.3 加权平均 vruntime 计算 (fair.c:715)

```c
u64 avg_vruntime(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    long weight = cfs_rq->sum_weight;   // 队列中所有实体的权重和
    s64 delta = 0;

    if (curr && !curr->on_rq)
        curr = NULL;

    if (weight) {
        s64 runtime = cfs_rq->sum_w_vruntime;  // 加权 vruntime 总和

        // 将当前运行实体（尚未入队）加入计算
        if (curr) {
            unsigned long w = scale_load_down(curr->load.weight);
            runtime += entity_key(cfs_rq, curr) * w;
            weight += w;
        }

        /* sign flips effective floor / ceiling
         *
         * 关键技巧：负数的除法在 C 语言中向 0 取整，
         * 这里通过减 (weight - 1) 实现向负无穷取整，
         * 使结果偏向左侧（更早的 vruntime），
         * 确保 avg_vruntime() + 0 一定产生合格的实体。
         */
        if (runtime < 0)
            runtime -= (weight - 1);

        delta = div_s64(runtime, weight);
    } else if (curr) {
        // 队列中只有一个实体时，它自己就是平均值
        delta = curr->vruntime - cfs_rq->zero_vruntime;
    }

    update_zero_vruntime(cfs_rq, delta);
    return cfs_rq->zero_vruntime;
}
```

**关键细节**:
- `zero_vruntime` 是一个基准偏移，防止 vruntime 溢出。所有 vruntime 都存储为相对于 `zero_vruntime` 的差值。
- 负数除法的取整方向被显式控制，保证合格性判断的准确性。

### 5.4 合格性判断 (fair.c:797)

```c
static int vruntime_eligible(struct cfs_rq *cfs_rq, u64 vruntime)
{
    struct sched_entity *curr = cfs_rq->curr;
    s64 avg = cfs_rq->sum_w_vruntime;   // 加权 vruntime 总和
    long load = cfs_rq->sum_weight;     // 权重和

    // 将当前运行实体加入
    if (curr && curr->on_rq) {
        unsigned long weight = scale_load_down(curr->load.weight);
        avg += entity_key(cfs_rq, curr) * weight;
        load += weight;
    }

    /* 判断: avg >= (vruntime - zero_vruntime) * load
     *
     * 等价于: avg/load >= vruntime - zero_vruntime
     * 等价于: avg_vruntime() >= vruntime
     * 等价于: V >= v_i → lag >= 0 → 合格
     *
     * 为什么不直接做除法？
     *   因为除法损失精度，用乘法比较更精确。
     */
    return avg >= vruntime_op(vruntime, "-", cfs_rq->zero_vruntime) * load;
}

int entity_eligible(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    return vruntime_eligible(cfs_rq, se->vruntime);
}
```

**关键优化**: 用乘法代替除法来避免精度损失。`avg >= (vruntime - zero) * load` 等价于 `avg/load >= vruntime - zero`，但前者没有除法舍入误差。

### 5.5 红黑树增强 (fair.c:853-909)

红黑树按 `vruntime` 排序，但每个节点增强了三个值：

```c
se->min_vruntime = min(se->vruntime, left->min_vruntime, right->min_vruntime)
se->min_slice    = min(se->slice, left->min_slice, right->min_slice)
se->max_slice    = max(se->slice, left->max_slice, right->max_slice)
```

**min_vruntime** 的作用：支持合格性剪枝。如果一棵子树的最小 vruntime 已经超过了合格线，整个子树都没有合格实体，可以直接跳过。

```c
static inline bool min_vruntime_update(struct sched_entity *se, bool exit)
{
    u64 old_min_vruntime = se->min_vruntime;
    u64 old_min_slice = se->min_slice;
    u64 old_max_slice = se->max_slice;
    struct rb_node *node = &se->run_node;

    se->min_vruntime = se->vruntime;
    __min_vruntime_update(se, node->rb_right);
    __min_vruntime_update(se, node->rb_left);

    se->min_slice = se->slice;
    __min_slice_update(se, node->rb_right);
    __min_slice_update(se, node->rb_left);

    se->max_slice = se->slice;
    __max_slice_update(se, node->rb_right);
    __max_slice_update(se, node->rb_left);

    // 返回是否所有值都没变
    return se->min_vruntime == old_min_vruntime &&
           se->min_slice == old_min_slice &&
           se->max_slice == old_max_slice;
}

RB_DECLARE_CALLBACKS(static, min_vruntime_cb, struct sched_entity,
                     run_node, min_vruntime, min_vruntime_update);
```

### 5.6 时间片保护 (RUN_TO_PARITY) (fair.c:960-989)

```c
static inline void update_protect_deadline(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    u64 slice = normalized_sysctl_sched_base_slice;
    u64 vprot = se->deadline;

    if (sched_feat(RUN_TO_PARITY))
        slice = cfs_rq_min_slice(cfs_rq);  // 用队列中最小时间片

    slice = min(slice, se->slice);
    if (slice != se->slice)
        vprot = min_vruntime(vprot, se->vruntime + calc_delta_fair(slice, se));

    se->vprot = vprot;  // 保护边界
}

static inline bool protect_slice(struct sched_entity *se)
{
    // 当前 vruntime 还在保护范围内 → 不允许抢占
    return vruntime_cmp(se->vruntime, "<", se->vprot);
}
```

**含义**: 当前任务在到达 `vprot` 之前不会被抢占。`vprot` 的计算：
- 基础值 = `deadline`（虚拟截止时间）
- 如果队列中有时间片更短的任务，`vprot` 会被提前到 `vruntime + min_slice`
- 这允许短时间片的任务更早地获得 CPU

### 5.7 pick_eevdf() —— 完整选择算法 (fair.c:1010)

```c
static struct sched_entity *pick_eevdf(struct cfs_rq *cfs_rq, bool protect)
{
    struct rb_node *node = cfs_rq->tasks_timeline.rb_root.rb_node;
    struct sched_entity *se = __pick_first_entity(cfs_rq);  // 最左节点
    struct sched_entity *curr = cfs_rq->curr;
    struct sched_entity *best = NULL;

    /* ── 快速路径: 只有一个实体 ── */
    if (cfs_rq->nr_queued == 1)
        return curr && curr->on_rq ? curr : se;

    /* ── 尝试 1: PICK_BUDDY ──
     * 如果 last-woken buddy 合格，选它（缓存局部性优化）
     * NEXT_BUDDY 特性会在唤醒时设置 cfs_rq->next
     */
    if (sched_feat(PICK_BUDDY) &&
        cfs_rq->next && entity_eligible(cfs_rq, cfs_rq->next)) {
        WARN_ON_ONCE(cfs_rq->next->sched_delayed);
        return cfs_rq->next;
    }

    /* ── 清理: 当前实体不在队列或不合格 ── */
    if (curr && (!curr->on_rq || !entity_eligible(cfs_rq, curr)))
        curr = NULL;

    /* ── 尝试 2: RUN_TO_PARITY 保护 ──
     * 如果当前任务还在保护范围内，让它继续运行
     */
    if (curr && protect && protect_slice(curr))
        return curr;

    /* ── 尝试 3: 最左节点（最早截止时间） ── */
    if (se && entity_eligible(cfs_rq, se)) {
        best = se;
        goto found;
    }

    /* ── 尝试 4: 堆搜索 ──
     * 在红黑树中搜索合格的、截止时间最早的实体
     * 时间复杂度: O(log n)
     */
    while (node) {
        struct rb_node *left = node->rb_left;

        /* 左子树有合格实体 → 进入左子树（截止时间的更早） */
        if (left && vruntime_eligible(cfs_rq,
                    __node_2_se(left)->min_vruntime)) {
            node = left;
            continue;
        }

        se = __node_2_se(node);

        /* 左子树为空或无合格实体 → 检查当前节点 */
        if (entity_eligible(cfs_rq, se)) {
            best = se;
            break;
        }

        /* 当前节点也不合格 → 进入右子树 */
        node = node->rb_right;
    }

found:
    /* 兜底: 如果没有找到更好的，返回当前实体 */
    if (!best || (curr && entity_before(curr, best)))
        best = curr;

    return best;
}
```

**决策流程图**:

```
pick_eevdf()
    │
    ├── nr_queued == 1? ──── Yes → 返回唯一实体
    │
    ├── PICK_BUDDY && next eligible? ── Yes → 返回 next buddy (缓存友好)
    │
    ├── curr eligible && protect_slice? ── Yes → 返回 curr (RUN_TO_PARITY)
    │
    ├── leftmost eligible? ── Yes → 返回 leftmost
    │
    └── heap search:
          ├── left subtree has eligible? → go left
          ├── current node eligible? → pick it, done
          └── go right
    │
    └── fallback → curr (if better than best)
```

### 5.8 update_curr() (fair.c:1281)

```c
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    struct rq *rq = rq_of(cfs_rq);
    s64 delta_exec;
    bool resched;

    if (unlikely(!curr))
        return;

    /* ① 计算执行时间 */
    delta_exec = update_se(rq, curr);
    if (unlikely(delta_exec <= 0))
        return;

    /* ② vruntime += delta_exec / weight */
    curr->vruntime += calc_delta_fair(delta_exec, curr);

    /* ③ 更新虚拟截止时间: vd = ve + slice/weight */
    resched = update_deadline(cfs_rq, curr);

    /* ④ 如果 DL Server 活跃，将执行时间计入 server */
    if (entity_is_task(curr)) {
        if (dl_server_active(&rq->fair_server))
            dl_server_update(&rq->fair_server, delta_exec);
    }

    /* ⑤ 带宽控制 */
    account_cfs_rq_runtime(cfs_rq, delta_exec);

    if (cfs_rq->nr_queued == 1)
        return;

    /* ⑥ 需要重调度或保护时间片耗尽 → 触发 resched */
    if (resched || !protect_slice(curr)) {
        resched_curr_lazy(rq);
        clear_buddies(cfs_rq, curr);
    }
}
```

### 5.9 update_deadline() (fair.c:1112)

```c
static bool update_deadline(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    /* 如果 vruntime 还没到达 deadline，不需要更新 */
    if (vruntime_cmp(se->vruntime, "<", se->deadline))
        return false;

    /* 使用自定义时间片或默认时间片 */
    if (!se->custom_slice)
        se->slice = sysctl_sched_base_slice;

    /* 核心公式: vd = ve + slice/weight */
    se->deadline = se->vruntime + calc_delta_fair(se->slice, se);
    avg_vruntime(cfs_rq);   // 刷新平均 vruntime

    return true;  // 时间片耗尽，需要重调度
}
```

### 5.10 实体放置 place_entity() (fair.c:5204)

新任务或唤醒任务入队时的 vruntime 放置：

```c
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    u64 vslice = calc_delta_fair(se->slice, se);
    u64 vruntime = avg_vruntime(cfs_rq);

    if (flags & PLACE_DEADLINE_INITIAL) {
        /* 新任务: 从 avg - 半个时间片 开始 */
        vruntime -= vslice / 2;
    }

    if (flags & PLACE_LAG) {
        /* 唤醒任务: 保留 lag = V - ve */
        /* 使 lag 在睡眠-唤醒周期中保持不变 */
        s64 lag = vruntime - se->vruntime;
        lag = clamp(lag, ...);  // 限制 lag 范围
        vruntime = avg_vruntime(cfs_rq) - lag;
    }

    se->vruntime = vruntime;
    se->deadline = se->vruntime + vslice;
}
```

### 5.11 EEVDF vs 传统 CFS 对比

| 维度 | 传统 CFS | EEVDF (6.18) |
|------|----------|-------------|
| **选择策略** | 取红黑树最左节点（最小 vruntime） | 搜索合格的、截止时间最早的实体 |
| **延迟保证** | 无显式截止时间 | 有虚拟截止时间 vd，延迟可预测 |
| **抢占模型** | 随时可能被更小 vruntime 抢占 | RUN_TO_PARITY 保护到平价点 |
| **睡眠公平** | 睡眠后 vruntime 落后，恢复后占优势 | PLACE_LAG 保持 lag 不变 |
| **时间片** | 隐式（由调度延迟决定） | 显式（se->slice），可自定义 |
| **红黑树增强** | 仅 min_vruntime | min_vruntime + min_slice + max_slice |

---

## 6. 唤醒抢占

### 6.1 wakeup_preempt() (core.c:2208)

```c
void wakeup_preempt(struct rq *rq, struct task_struct *p, int flags)
{
    struct task_struct *donor = rq->donor;

    if (p->sched_class == donor->sched_class)
        // 同类: 调用该类的 wakeup_preempt 回调
        donor->sched_class->wakeup_preempt(rq, p, flags);
    else if (sched_class_above(p->sched_class, donor->sched_class))
        // 高优先级类: 直接设置 resched
        resched_curr(rq);

    /* 优化: 如果需要 resched，跳过下一次 clock update */
    if (task_on_rq_queued(donor) && test_tsk_need_resched(rq->curr))
        rq_clock_skip_update(rq);
}
```

### 6.2 check_preempt_wakeup_fair() (fair.c:8862)

```c
static void check_preempt_wakeup_fair(struct rq *rq, struct task_struct *p, int wake_flags)
{
    enum preempt_wakeup_action preempt_action = PREEMPT_WAKEUP_PICK;
    struct task_struct *donor = rq->donor;
    struct sched_entity *nse, *se = &donor->se, *pse = &p->se;

    if (unlikely(se == pse))
        return;

    if (task_is_throttled(p))
        return;

    if (test_tsk_need_resched(rq->curr))
        return;

    if (!sched_feat(WAKEUP_PREEMPTION))
        return;

    find_matching_se(&se, &pse);

    cse_is_idle = se_is_idle(se);
    pse_is_idle = se_is_idle(pse);

    /* ── 情况 1: 当前空闲，唤醒者非空闲 → 抢占 ── */
    if (cse_is_idle && !pse_is_idle) {
        preempt_action = PREEMPT_WAKEUP_SHORT;  // 不给空闲任务切片保护
        goto preempt;
    }

    if (cse_is_idle != pse_is_idle)
        return;

    /* ── 情况 2: BATCH/IDLE 任务不抢占 ── */
    if (unlikely(!normal_policy(p->policy)))
        return;

    update_curr(cfs_rq);

    /* ── 情况 3: PREEMPT_SHORT ──
     * 唤醒者有更短时间片 → 取消当前任务的 RUN_TO_PARITY 保护
     */
    if (sched_feat(PREEMPT_SHORT) && (pse->slice < se->slice)) {
        preempt_action = PREEMPT_WAKEUP_SHORT;
        goto pick;
    }

    /* ── 情况 4: WF_FORK / sched_delayed → 不抢占 ── */
    if ((wake_flags & WF_FORK) || pse->sched_delayed)
        return;

    /* ── 情况 5: NEXT_BUDDY ── */
    if (sched_feat(NEXT_BUDDY) &&
        set_preempt_buddy(cfs_rq, wake_flags, pse, se)) {
        if (wake_flags & WF_SYNC)
            preempt_action = preempt_sync(rq, wake_flags, pse, se);
    }

    switch (preempt_action) {
    case PREEMPT_WAKEUP_NONE:
        return;
    case PREEMPT_WAKEUP_RESCHED:
        goto preempt;
    case PREEMPT_WAKEUP_SHORT:
        // fallthrough
    case PREEMPT_WAKEUP_PICK:
        break;
    }

pick:
    /* 唤醒者优先级更高 → 设置 next buddy 用于下次 pick */
    set_next_buddy(pse);

preempt:
    resched_curr_lazy(rq);
}
```

**抢占决策总结**:

| 条件 | 结果 |
|------|------|
| 当前 idle + 唤醒者非 idle | 立即抢占，无保护 |
| 唤醒者更短时间片 (PREEMPT_SHORT) | 取消 RUN_TO_PARITY，允许抢占 |
| WF_FORK | 不抢占 |
| sched_delayed | 不抢占（违反 EEVDF） |
| NEXT_BUDDY + 合适 | 设置 next buddy，可能抢占 |
| 其他 | 默认 PICK：设置 buddy，延迟抢占 |

---

## 7. Tick 驱动路径

### 7.1 sched_tick() (core.c:5579)

```c
void sched_tick(void)
{
    int cpu = smp_processor_id();
    struct rq *rq = cpu_rq(cpu);
    struct task_struct *donor;
    struct rq_flags rf;
    unsigned long hw_pressure;

    /* 频率缩放 */
    if (housekeeping_cpu(cpu, HK_TYPE_KERNEL_NOISE))
        arch_scale_freq_tick();

    sched_clock_tick();
    rq_lock(rq, &rf);
    donor = rq->donor;

    psi_account_irqtime(rq, donor, NULL);
    update_rq_clock(rq);
    hw_pressure = arch_scale_hw_pressure(cpu_of(rq));
    update_hw_load_avg(rq_clock_task(rq), rq, hw_pressure);

    /* lazy resched → 提升为 eager */
    if (dynamic_preempt_lazy() && tif_test_bit(TIF_NEED_RESCHED_LAZY))
        resched_curr(rq);

    /* 调用当前调度类的 tick 处理 */
    donor->sched_class->task_tick(rq, donor, 0);

    /* 全局负载统计 */
    calc_global_load_tick(rq);
    sched_core_tick(rq);
    task_tick_mm_cid(rq, donor);
    scx_tick(rq);

    rq_unlock(rq, &rf);

    if (donor->flags & PF_WQ_WORKER)
        wq_worker_tick(donor);

    /* 负载均衡触发 */
    if (!scx_switched_all()) {
        rq->idle_balance = idle_cpu(cpu);
        sched_balance_trigger(rq);
    }
}
```

### 7.2 task_tick_fair() (fair.c:13476)

```c
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &curr->se;

    /* 遍历调度实体层次（cgroup 场景） */
    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);
        entity_tick(cfs_rq, se, queued);
    }

    /* NUMA 平衡 */
    if (static_branch_unlikely(&sched_numa_balancing))
        task_tick_numa(rq, curr);

    update_misfit_status(curr, rq);
    check_update_overutilized_status(task_rq(curr));
    task_tick_core(rq, curr);
}
```

### 7.3 entity_tick() (fair.c:5623)

```c
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    /* ① 更新当前任务的运行时统计 */
    update_curr(cfs_rq);

    /* ② 更新 PELT 负载平均值 */
    update_load_avg(cfs_rq, curr, UPDATE_TG);
    update_cfs_group(curr);

    /* ③ HRTICK: 精确时间片计时 */
#ifdef CONFIG_SCHED_HRTICK
    if (queued) {
        resched_curr_lazy(rq_of(cfs_rq));
        return;
    }
#endif
}
```

### 7.4 resched_curr_lazy()

`resched_curr_lazy()` 设置 `TIF_NEED_RESCHED_LAZY` 而非立即设置 `TIF_NEED_RESCHED`。lazy 标志会在 tick 中被提升为 eager（见 7.1），这减少了不必要的立即抢占，提高了吞吐量。

---

## 8. Deadline 调度

### 8.1 数据结构

**sched_dl_entity** (`include/linux/sched.h:644`):

```c
struct sched_dl_entity {
    struct rb_node    rb_node;           // 按 deadline 排序的红黑树节点
    u64               dl_runtime;        // 每个周期最大运行时间
    u64               dl_deadline;       // 相对截止时间
    u64               dl_period;         // 周期（与 deadline 可不同）
    u64               dl_bw;             // 带宽 = dl_runtime / dl_period
    u64               dl_density;        // 密度 = dl_runtime / dl_deadline
    s64               runtime;           // 剩余运行时间（可 < 0 = 超限）
    u64               deadline;          // 绝对截止时间
    unsigned int      flags;
    // 标志: dl_throttled, dl_yielded, dl_non_contending, dl_overrun,
    //       dl_server, dl_defer, dl_defer_armed, dl_server_active, dl_defer_running
};
```

**dl_rq** (`sched.h:862`):

```c
struct dl_rq {
    struct rb_root_cached  root;               // 按 deadline 排序的红黑树
    unsigned int           dl_nr_running;
    struct { u64 curr; u64 next; } earliest_dl; // 缓存最早/次早 deadline
    struct rb_root_cached  pushable_dl_tasks_root;
    u64                    running_bw;          // 活跃利用率
    u64                    this_bw;             // 分配的带宽
};
```

### 8.2 EDF 调度

红黑树按**绝对截止时间**排序，选择最左节点：经典 Earliest Deadline First。

### 8.3 CBS 运行时执行

```c
dl_se->runtime -= scaled_delta_exec;       // 消耗运行时间

if (dl_runtime_exceeded(dl_se) || dl_se->dl_yielded) {
    dl_se->dl_throttled = 1;               // 节流
    dequeue_dl_entity(dl_se, 0);           // 移出队列
    // hrtimer 在下一个周期触发补充
}
```

### 8.4 补充机制

```c
replenish_dl_entity() {
    /* 如果超限，前进截止时间直到运行时间非负 */
    while (dl_se->runtime <= 0) {
        dl_se->deadline += dl_se->dl_period;
        dl_se->runtime += dl_se->dl_runtime;
    }
}
```

### 8.5 准入控制

`dl_bw`（per-root-domain）跟踪总分配带宽。可调度性测试：

```
sum(dl_runtime / dl_period) ≤ available_bandwidth
```

---

## 9. DL Server 状态机

DL Server 是 6.18 的新特性，将 CFS 整体包装在一个 deadline server 中，赋予 CFS 带宽保证。

### 9.1 状态转换图 (deadline.c:1582)

```
                                        6
                            +--------------------+
                            v                    |
     +-------------+  4   +-----------+  5     +------------------+
 +-> |   A:init    | <--- | D:running | -----> | E:replenish-wait |
 |   +-------------+      +-----------+        +------------------+
 |     |         |    1     ^    ^               |
 |     | 1       +----------+    | 3             |
 |     v                         |               |
 |   +--------------------------------+   2      |
 |   |                                | ----+    |
 | 8 |       B:zero_laxity-wait       |     |    |
 |   |                                | <---+    |
 |   +--------------------------------+          |
 |     |              ^     ^           2        |
 |     | 7            | 2   +--------------------+
 |     v              |
 |   +-------------+  |
 +-- | C:idle-wait | -+
     +-------------+
       ^ 7       |
       +---------+
```

### 9.2 状态定义

| 状态 | dl_server_active | dl_throttled | dl_defer_armed | dl_defer_running | dl_defer_idle |
|------|------------------|-------------|----------------|------------------|---------------|
| **[A] init** | 0 | 0 | 0 | 0/1 | 0 |
| **[B] zero_laxity-wait** | 1 | 1 | 1 | 0 | 0 |
| **[C] idle-wait** | 1 | 1 | 1 | 0 | 1 |
| **[D] running** | 1 | 0 | 0 | 1 | 0 |
| **[E] replenish-wait** | 1 | 1 | 0 | 1 | 0 |

### 9.3 状态转换

| 转换 | 触发条件 | 说明 |
|------|----------|------|
| A→B | `dl_server_start()` + defer | 进入零松弛度等待 |
| A→D | `dl_server_start()` + 有运行时间 | 直接开始运行 |
| B→D | 到达零松弛度时间点 | server 被激活 |
| B→E | 运行时间耗尽 | 进入补充等待 |
| C→D | CPU 从空闲唤醒 | server 恢复运行 |
| D→E | 运行时间耗尽 | 补充等待 |
| E→A | 补充完成 | 回到 init |
| D→B | 进入零松弛度 | 再次等待 |

### 9.4 核心代码 (deadline.c:1575)

```c
void dl_server_update(struct sched_dl_entity *dl_se, s64 delta_exec)
{
    // 0 runtime = fair server 关闭
    if (dl_se->dl_runtime)
        update_curr_dl_se(dl_se->rq, dl_se, delta_exec);
}
```

在 `update_curr()` 中：

```c
/* 如果 DL Server 活跃，将 CFS 执行时间计入 server */
if (dl_server_active(&rq->fair_server))
    dl_server_update(&rq->fair_server, delta_exec);
```

---

## 10. 实时调度

### 10.1 rt_prio_array (sched.h:309)

```c
struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1);   // 位图，O(1) 找最高优先级
    struct list_head queue[MAX_RT_PRIO];      // 每个优先级一个 FIFO 队列
};
```

### 10.2 选择 (rt.c:1665)

```c
idx = sched_find_first_bit(array->bitmap);    // O(1) 位图扫描
queue = array->queue + idx;
next = list_entry(queue->next, ...);           // FIFO 队列头部
```

### 10.3 RR 时间片耗尽 (rt.c:2511)

```c
task_tick_rt() {
    if (--p->rt.time_slice <= 0) {
        p->rt.time_slice = sched_rr_timeslice;  // 重新分配时间片
        /* 将任务移到同优先级队列末尾 */
        list_move_tail(&p->rt.run_list, &array->queue[idx]);
    }
}
```

### 10.4 Push/Pull 迁移

| 操作 | 触发 | 函数 |
|------|------|------|
| **Push** | RT 任务唤醒并抢占低优先级任务 | `push_rt_task()` (rt.c:1933) |
| **Pull** | RT 任务阻塞 | `pull_rt_task()` (rt.c:2234) |
| **RT_PUSH_IPI** | IPI 减少锁竞争 | 特性开关 |

### 10.5 RT 带宽

```
sysctl_sched_rt_period  = 1,000,000 μs (1秒)
sysctl_sched_rt_runtime = 950,000 μs     (95%)
```

防止 RT 任务饿死非 RT 工作负载。

---

## 11. SCHED_PROXY_EXEC 代理执行

### 11.1 问题与解决方案

**问题**: 高优先级任务被互斥锁阻塞时让出 CPU，锁持有者可能优先级很低 → 优先级反转。

**解决方案**: 阻塞任务留在队列上，调度器跟随 `blocked_on` 链找到锁持有者，代理执行。

### 11.2 find_proxy_task() (core.c:6620)

```c
static struct task_struct *
find_proxy_task(struct rq *rq, struct task_struct *donor, struct rq_flags *rf)
{
    struct task_struct *owner = NULL;
    int this_cpu = cpu_of(rq);
    struct task_struct *p;
    struct mutex *mutex;

    /* 跟随 blocked_on 链:
     *   task->blocked_on -> mutex->owner -> task...
     *
     * 锁顺序: p->pi_lock → rq->lock → mutex->wait_lock
     */
    for (p = donor; task_is_blocked(p); p = owner) {
        mutex = p->blocked_on;
        if (!mutex)
            return NULL;  // 链中发生了变化，重新选择

        guard(raw_spinlock)(&mutex->wait_lock);

        /* 双重检查: wait_lock 持有期间确认仍被阻塞 */
        if (mutex != __get_task_blocked_on(p))
            return NULL;  // 重新选择

        owner = __mutex_owner(mutex);
        if (!owner) {
            __clear_task_blocked_on(p, mutex);
            return p;     // 锁无持有者，p 可以继续
        }

        /* ── 当前不支持的情况 ── */
        if (!READ_ONCE(owner->on_rq) || owner->se.sched_delayed)
            return proxy_deactivate(rq, donor);  // 所有者不在队列或延迟出队

        if (task_cpu(owner) != this_cpu)
            return proxy_deactivate(rq, donor);  // 所有者在其他 CPU（暂不支持跨 CPU 代理）

        if (task_on_rq_migrating(owner))
            return proxy_resched_idle(rq);       // 迁移中，先 idle 等待

        /* 双重检查: 仍在当前 CPU 且不在迁移中 */
        if (!task_on_rq_queued(owner) || task_cpu(owner) != this_cpu)
            return NULL;  // 重新选择

        /* 自引用检测: owner == p (可能的 unlock 竞争) */
        if (owner == p)
            return proxy_resched_idle(rq);
    }

    WARN_ON_ONCE(owner && !owner->on_rq);
    return owner;  // 返回可运行的锁持有者
}
```

### 11.3 当前限制

从代码注释 (`XXX`) 可以看出：

- **不支持跨 CPU 代理**: `if (task_cpu(owner) != this_cpu) → deactivate`
- **不支持阻塞的所有者**: `if (!owner->on_rq || owner->sched_delayed) → deactivate`
- 不支持延迟出队的任务

这些限制意味着 proxy exec 目前只能在同一 CPU 上、锁持有者可运行的情况下生效。

### 11.4 proxy_tag_curr() (core.c:6734)

```c
static inline void proxy_tag_curr(struct rq *rq, struct task_struct *owner)
{
    if (!sched_proxy_exec())
        return;

    /* donor 和 owner 形成一个 push/pull 原子对。
     * 确保 owner 不可被 push/pull。
     * 只能通过 dequeue/enqueue 循环实现。
     */
    dequeue_task(rq, owner, DEQUEUE_NOCLOCK | DEQUEUE_SAVE);
    enqueue_task(rq, owner, ENQUEUE_NOCLOCK | ENQUEUE_RESTORE);
}
```

---

## 12. 负载均衡

### 12.1 调度域层次

```
SMT（超线程兄弟） → CORE → DIE/Socket → NUMA 节点 → NUMA 系统
```

每个 `sched_domain` 包含 parent/child 链、sched_group 列表、balance_interval、flags。

### 12.2 sched_balance_rq() (fair.c:11920)

```c
static int sched_balance_rq(int this_cpu, struct rq *this_rq,
                            struct sched_domain *sd, enum cpu_idle_type idle,
                            int *continue_balancing)
{
    struct lb_env env = {
        .sd         = sd,
        .dst_cpu    = this_cpu,
        .dst_rq     = this_rq,
        .idle       = idle,
        .loop_break = SCHED_NR_MIGRATE_BREAK,
        .tasks      = LIST_HEAD_INIT(env.tasks),
    };

redo:
    if (!should_we_balance(&env))
        goto out_balanced;

    /* SD_SERIALIZE: 一次只有一个 CPU 执行平衡 */
    if (sd->flags & SD_SERIALIZE) {
        if (!atomic_try_cmpxchg_acquire(&sched_balance_running, &zero, 1))
            goto out_balanced;
    }

    /* ── 找到最忙的组 ── */
    group = sched_balance_find_src_group(&env);
    if (!group) goto out_balanced;

    /* ── 找到最忙的 runqueue ── */
    busiest = sched_balance_find_src_rq(&env, group);
    if (!busiest) goto out_balanced;

    /* ── 计算不平衡量 ── */
    update_lb_imbalance_stat(&env, sd, idle);

    /* ── 迁移任务 ── */
    env.src_cpu = busiest->cpu;
    env.src_rq = busiest;

    if (busiest->nr_running > 1) {
        env.loop_max = min(sysctl_sched_nr_migrate, busiest->nr_running);

more_balance:
        rq_lock_irqsave(busiest, &rf);
        update_rq_clock(busiest);

        /* 从最忙的 CPU 分离任务 */
        cur_ld_moved = detach_tasks(&env);

        /* 分离后解锁（任务标记 TASK_ON_RQ_MIGRATING，安全） */
        rq_unlock(busiest, &rf);

        if (cur_ld_moved) {
            attach_tasks(&env);       // 附加到目标 CPU
            ld_moved += cur_ld_moved;
        }

        /* 需要中断恢复 → 重新尝试 */
        if (env.flags & LBF_NEED_BREAK) {
            // ...
        }

        /* 如果仍然不平衡 → 继续迁移 */
        if (!sched_balance_stopped() && env.loop < env.loop_max)
            goto more_balance;
    }

    /* ── 主动负载均衡 ── */
    if (ld_moved == 0 && idle == CPU_NOT_IDLE && busiest->nr_running > 1)
        active_balance = active_load_balance_cpu_stop(this_cpu, sd);

out_balanced:
    return ld_moved;
}
```

### 12.3 三种平衡触发

| 类型 | 触发 | 函数 | 策略 |
|------|------|------|------|
| **周期性** | sched_tick → sched_balance_trigger() | `sched_balance_domains()` (fair.c:12386) | 从低层到高层遍历域树 |
| **空闲** | CPU 即将空闲 | `sched_balance_newidle()` (fair.c:12970) | 激进搜索 |
| **NOHZ** | tickless 空闲 | `nohz_idle_balance()` (fair.c:12884) | idle load balancer 代表所有空闲 CPU |

### 12.4 detach_tasks / attach_tasks

```c
detach_tasks(&env):
    /* 从最忙 CPU 的任务列表中分离可迁移的任务 */
    /* 根据 env.idle 决定迁移策略:
     *   CPU_IDLE    → 更激进，迁移任何任务
     *   CPU_NOT_IDLE → 保守，只迁移确实需要迁移的
     */

attach_tasks(&env):
    /* 将分离的任务附加到目标 CPU */
    /* 调用 activate_task() 入队 */
```

---

## 13. PELT 负载跟踪

### 13.1 sched_avg

每个 `sched_entity` 和 `cfs_rq` 维护：

```c
struct sched_avg {
    u32 load_avg;       // 衰减后的负载贡献
    u32 util_avg;       // CPU 利用率估算
    u32 runnable_avg;   // 可运行时间比例
};
```

### 13.2 衰减机制

- 使用指数衰减平均，**半衰期 1024μs**
- 每个 PELT 周期（~1ms）更新一次
- 提供 sub-tick 粒度的负载跟踪
- 驱动 CPU 频率调节（schedutil cpufreq governor）

### 13.3 更新路径

```
entity_tick()
  → update_load_avg(cfs_rq, curr, UPDATE_TG)
    → __update_load_avg()
      → decay_load()    // 按时间衰减
      → accumulate_sum() // 累加当前周期贡献
```

---

## 14. 唤醒路径

### 14.1 try_to_wake_up() (core.c:4148)

```c
try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
    /* ① 状态匹配检查 */
    if (!ttwu_state_match(p, state, &success))
        goto out;

    /* ② 如果任务在其他 CPU 上运行 → 远程 IPI */
    if (task_cpu(p) != this_cpu && ttwu_queue_wakelist(p, task_cpu(p), wake_flags))
        goto out;

    /* ③ 选择目标 CPU */
    cpu = select_task_rq(p, this_cpu, prev_cpu, wake_flags);

    /* ④ 入队 */
    enqueue_task(rq, p, ENQUEUE_WAKEUP | ENQUEUE_NOCLOCK);

    /* ⑤ 如果目标 CPU 空闲或任务优先级更高 → 触发 resched */
    if (cpu != this_cpu) {
        resched_curr(rq);
        ttwu_queue_remote(p, cpu, wake_flags);  // IPI
    }

    /* ⑥ 唤醒抢占 */
    wakeup_preempt(rq, p, wake_flags);
}
```

### 14.2 select_task_rq() — CPU 选择

```
select_task_rq()
  → p->sched_class->select_task_rq(p, prev_cpu, wake_flags)
    → 遍历调度域树:
      ├── SD_WAKE_AFFINE: 偏好唤醒 CPU 或之前 CPU
      ├── 计算每个候选 CPU 的负载
      └── 选择亲和域内负载最轻的 CPU
```

---

## 15. 调度器特性开关

定义在 `kernel/sched/features.h`，可通过 `/sys/kernel/debug/sched_features` 运行时切换。

| 特性 | 默认 | 描述 |
|------|------|------|
| `PLACE_LAG` | ✓ | EEVDF lag 保持，跨睡眠/唤醒周期公平 |
| `PLACE_DEADLINE_INITIAL` | ✓ | 新任务从半个时间片开始 |
| `PLACE_REL_DEADLINE` | ✓ | 迁移时保持相对截止时间不变 |
| `RUN_TO_PARITY` | ✓ | 保护当前任务直到 lag=0 或时间片耗尽 |
| `PREEMPT_SHORT` | ✓ | 短时间片唤醒者可取消 RUN_TO_PARITY 保护 |
| `NEXT_BUDDY` | ✗ | 偏好最后唤醒的任务（缓存局部性） |
| `PICK_BUDDY` | ✓ | 允许在特定情况下忽略 cfs_rq->next |
| `CACHE_HOT_BUDDY` | ✓ | 减少缓存热任务的迁移 |
| `DELAY_DEQUEUE` | ✓ | 延迟出队，让非合格任务消耗负 lag |
| `DELAY_ZERO` | ✓ | 出队/唤醒时将 lag 截断到 0 |
| `WAKEUP_PREEMPTION` | ✓ | 允许唤醒时抢占 |
| `UTIL_EST` | ✓ | 利用率估算，驱动频率调节 |
| `TTWU_QUEUE` | ✓¹ | 远程唤醒通过 IPI 排队 |
| `RT_PUSH_IPI` | ✓ | RT push 迁移使用 IPI |
| `NI_RANDOM` | ✓ | 按成功率随机化 newidle 平衡 |
| `SIS_UTIL` | ✓ | 唤醒时限制 LLC 域扫描范围 |
| `HRTICK` | ✗ | 高精度 tick（精确时间片） |
| `HRTICK_DL` | ✗ | Deadline 高精度 tick |
| `LATENCY_WARN` | ✗ | 延迟警告 |

¹ 在 PREEMPT_RT 内核中默认为关闭。

---

## 16. 关键数据结构关系

```
struct rq (per-CPU)
  │
  ├── struct cfs_rq ──────────────────────────────┐
  │     ├── tasks_timeline (RB-tree, 按 vruntime) │
  │     │   └── struct sched_entity ◄─────────────┼──┐
  │     │         ├── vruntime                    │  │
  │     │         ├── deadline (vd = ve + r/w)    │  │
  │     │         ├── slice                       │  │
  │     │         ├── vprot (保护边界)             │  │
  │     │         ├── min_vruntime (子树最小)      │  │
  │     │         ├── min_slice / max_slice       │  │
  │     │         └── struct sched_avg (PELT)     │  │
  │     ├── sum_w_vruntime                        │  │
  │     ├── sum_weight                            │  │
  │     ├── curr / next                           │  │
  │     └── load                                  │  │
  │                                               │  │
  ├── struct rt_rq ───────────────────────────────┤  │
  │     └── rt_prio_array                         │  │
  │           ├── bitmap (O(1) 优先级查找)         │  │
  │           └── queue[MAX_RT_PRIO] (FIFO)       │  │
  │                                               │  │
  ├── struct dl_rq ───────────────────────────────┤  │
  │     └── root (RB-tree, 按 deadline) ──────────┼──┤
  │           └── struct sched_dl_entity ◄────────┼──┤
  │                                               │  │
  ├── struct scx_rq (BPF 可扩展，可选)             │  │
  │                                               │  │
  ├── struct sched_dl_entity fair_server ─────────┘  │ (DL Server 包装 CFS)
  │                                                  │
  ├── struct task_struct *curr      (执行上下文)     │
  ├── struct task_struct *donor     (调度上下文)     │
  ├── struct task_struct *idle      (空闲任务)       │
  ├── struct task_struct *stop      (停机任务)       │
  ├── struct sched_domain *sd       (调度域树)       │
  │                                                  │
  └── clock / clock_task / clock_pelt                │
                                                     │
struct task_struct ◄─────────────────────────────────┘
  ├── struct sched_entity se        (CFS 调度实体)
  ├── struct sched_rt_entity rt     (RT 调度实体)
  ├── struct sched_dl_entity dl     (Deadline 调度实体)
  ├── const struct sched_class *sched_class
  ├── int policy                    (NORMAL, FIFO, RR, DEADLINE, ...)
  ├── int prio, static_prio, normal_prio
  ├── unsigned long blocked_on      (互斥锁指针, proxy exec 用)
  └── int on_rq                     (是否在运行队列上)
```

---

## 17. 本版本架构创新

### 17.1 EEVDF 替代 CFS vruntime

| 方面 | 变化 |
|------|------|
| 选择算法 | 从"最小 vruntime 优先"变为"合格 + 最早虚拟截止时间" |
| 延迟保证 | 引入虚拟截止时间 vd，延迟可预测 |
| 红黑树 | 从单增强 (min_vruntime) 到三增强 (min_vruntime, min_slice, max_slice) |
| 时间片 | 从隐式（由调度延迟决定）到显式 (se->slice) |

### 17.2 RUN_TO_PARITY / 时间片保护

保护当前任务不被抢占，直到它到达平价点（lag = 0）或耗尽时间片。减少上下文切换抖动，同时保持公平性。`PREEMPT_SHORT` 允许短时间片唤醒者取消此保护。

### 17.3 SCHED_PROXY_EXEC

调度器级优先级继承。阻塞任务留在队列上，执行代理给锁持有者。当前版本限制：仅支持同 CPU 代理，不支持跨 CPU 和阻塞链。

### 17.4 DL Server 包装 CFS

CFS 任务整体获得 deadline 语义的带宽保证。Server 运行在零松弛度（zero-laxity）点，有完整的 5 状态状态机。

### 17.5 sched_ext (SCX)

BPF 可编程调度类，用户空间可定义自定义调度策略。条件激活，不在 `for_each_active_class` 默认路径中。

### 17.6 DELAY_DEQUEUE

任务不立即从 RB-tree 中移除，允许非合格任务消耗负 lag。`DELAY_ZERO` 在出队时将 lag 截断到 0。

---

## 附录: 关键函数索引

| 函数 | 文件:行 | 作用 |
|------|---------|------|
| `schedule()` | core.c:7022 | 调度入口 |
| `__schedule()` | core.c:6790 | 核心调度函数 |
| `pick_next_task()` | core.c:5956 | 选择下一个任务 |
| `context_switch()` | core.c:5275 | 上下文切换 |
| `pick_eevdf()` | fair.c:1010 | EEVDF 选择算法 |
| `avg_vruntime()` | fair.c:715 | 加权平均 vruntime |
| `vruntime_eligible()` | fair.c:797 | 合格性判断 |
| `update_curr()` | fair.c:1281 | 更新当前任务运行时 |
| `update_deadline()` | fair.c:1112 | 更新虚拟截止时间 |
| `place_entity()` | fair.c:5204 | 实体放置（入队时 vruntime 设置） |
| `entity_tick()` | fair.c:5623 | tick 处理 |
| `task_tick_fair()` | fair.c:13476 | CFS tick 入口 |
| `check_preempt_wakeup_fair()` | fair.c:8862 | 唤醒抢占 |
| `wakeup_preempt()` | core.c:2208 | 唤醒抢占分发 |
| `sched_tick()` | core.c:5579 | 调度 tick 入口 |
| `find_proxy_task()` | core.c:6620 | 代理执行：找锁持有者 |
| `sched_balance_rq()` | fair.c:11920 | 核心负载均衡 |
| `try_to_wake_up()` | core.c:4148 | 唤醒路径 |
| `dl_server_update()` | deadline.c:1575 | DL Server 更新 |

---

*文档完。*
