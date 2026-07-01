# Linux 6.18.34 Spinlock 代码实现全景

## 一、四层架构

Spinlock 的实现分为 **4 层**，从用户 API 到底层汇编：

```
spin_lock()                  ← 驱动层 API (include/linux/spinlock.h)
  ↓
_raw_spin_lock()             ← SMP 封装层 (include/linux/spinlock_api_smp.h / kernel/locking/spinlock.c)
  ↓
do_raw_spin_lock()           ← 原始操作层 (include/linux/spinlock.h)
  ↓
arch_spin_lock()             ← 架构特定层 (arch/x86/include/asm/ → qspinlock)
```

## 二、数据结构

### 2.1 `spinlock_t` — 驱动层类型

```
spinlock_t
  └── raw_spinlock_t rlock
```

### 2.2 `raw_spinlock_t` — 核心类型

位置：`include/linux/spinlock_types_raw.h:14`

```c
typedef struct raw_spinlock {
    arch_spinlock_t raw_lock;       // 架构特定实现
#ifdef CONFIG_DEBUG_SPINLOCK
    unsigned int magic, owner_cpu;  // 调试: magic = 0xdead4ead
    void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;     // lockdep 集成
#endif
} raw_spinlock_t;
```

### 2.3 `arch_spinlock_t` / `qspinlock` — x86/arm64 上的实际实现

位置：`include/asm-generic/qspinlock_types.h:14`

一个 **32 位原子值**，按位域划分：

| 位     | 含义         | 说明                           |
|--------|-------------|-------------------------------|
| 0-7    | locked      | 锁状态 (0=空闲, 1=持有)         |
| 8      | pending     | 待定位，避免第一个竞争者排队     |
| 9-17   | 保留/未用    |                               |
| 18-19  | tail index  | 嵌套层索引 (0-3: task/softirq/hardirq/NMI) |
| 20-31  | tail cpu    | 队列尾部 CPU 编号              |

```c
typedef struct qspinlock {
    union {
        atomic_t val;
        struct { u8 locked; u8 pending; };   // 小端序
        struct { u16 locked_pending; u16 tail; };
    };
} arch_spinlock_t;
```

### 2.4 MCS 队列节点

位置：`kernel/locking/mcs_spinlock.h`

```c
struct mcs_spinlock {
    struct mcs_spinlock *next;   // 指向队列中的下一个节点
    int locked;                  // 1 = 获取到锁, 0 = 等待
    int count;                   // 嵌套计数器
};
```

Per-CPU 节点数组（最多 4 个嵌套级别）：
```c
static DEFINE_PER_CPU_ALIGNED(struct qnode, qnodes[_Q_MAX_NODES]);
```

## 三、加锁流程 — `queued_spin_lock`

位置：`include/asm-generic/qspinlock.h:107`

### 快速路径 (无竞争)

```c
static __always_inline void queued_spin_lock(struct qspinlock *lock)
{
    int val = 0;
    // 原子 CAS: 0 -> 1 (locked)
    if (likely(atomic_try_cmpxchg_acquire(&lock->val, &val, _Q_LOCKED_VAL)))
        return;  // 成功，直接返回
    queued_spin_lock_slowpath(lock, val);  // 进入慢速路径
}
```

### 慢速路径 (有竞争)

位置：`kernel/locking/qspinlock.c:130`

**状态转换图：**

```
uncontended   (0,0,0) → (0,0,1) ──────────────────────→ (*,*,0)  unlock
                            │ ^--------.------.
                            v           \      \
pending        (0,1,1) +→ (0,1,0)       \       |
                            │ ^--'        |       |
                            v             |       |
uncontended    (n,x,y) +→ (n,0,0) ───────'       |
  queue         │ ^--'                           |
                v                                |
contended       (*,x,y) +→ (*,0,0) → (*,0,1) ───'
  queue          ^--'
```

其中 `(tail, pending, locked)`。

**步骤详解：**

1. **Pending 等待** (L150-L154)
   - 如果看到 `(0,1,0)` 状态，最多循环 `_Q_PENDING_LOOPS = 512` 次等待 owner 释放
   - 保证前向推进，不会无限等待

2. **Pending 抢占** (L167-L206)
   - 设置 pending 位 (`btsl` 指令)，尝试以 pending 方式获取锁
   - 如果检测到竞争则撤销 pending 并进入排队
   - 如果没有竞争但 lock 被持有，等待 lock 释放后通过 `clear_pending_set_locked()` 获取
   - 这是避免第一个竞争者排队的优化

3. **MCS 排队** (L212-L302)
   - 分配 per-CPU 节点 (`qnodes[4]`，对应 4 个嵌套级别)
   - 通过 `xchg_tail()` 将自己发布到队列尾部
   - 如果有前驱，链接到队列 (`prev->next = node`)
   - 等待前驱节点唤醒 (`arch_mcs_spin_lock_contended` → `smp_cond_load_acquire`)

4. **获取锁** (L330-L370)
   - 到达队列头部后，等待 owner 和 pending 位都清零
   - 如果是队列中唯一节点，直接 CAS 获取锁（无竞争路径）
   - 否则调用 `set_locked()` 获取锁
   - 如果有后续节点，传递锁 (`next->locked = 1`)

### 解锁流程

位置：`include/asm-generic/qspinlock.h:123`

```c
static __always_inline void queued_spin_unlock(struct qspinlock *lock)
{
    smp_store_release(&lock->locked, 0);  // 释放语义的 store
}
```

- 使用 `smp_store_release` 提供 release 语义，确保临界区内的所有操作在解锁前完成
- 如果有等待者，当前驱的 `node->locked` 被设置为 1 时，等待者被唤醒

## 四、MCS 锁基础

位置：`kernel/locking/mcs_spinlock.h`

qspinlock 基于 **MCS (Mellor-Crummey & Scott)** 锁：

```c
void mcs_spin_lock(struct mcs_spinlock **lock, struct mcs_spinlock *node)
{
    node->locked = 0;
    node->next = NULL;
    prev = xchg(lock, node);        // 原子交换，把自己挂到队列
    if (prev == NULL) return;       // 队列为空，直接获取
    WRITE_ONCE(prev->next, node);   // 链接到前驱
    arch_mcs_spin_lock_contended(&node->locked);  // 自旋等待
}

void mcs_spin_unlock(struct mcs_spinlock **lock, struct mcs_spinlock *node)
{
    struct mcs_spinlock *next = READ_ONCE(node->next);

    if (likely(!next)) {
        // 没有后续节点，尝试 CAS 清空队列
        if (likely(cmpxchg_release(lock, node, NULL) == node))
            return;
        // 有后续节点正在链接，等待它设置 next
        while (!(next = READ_ONCE(node->next)))
            cpu_relax();
    }
    // 传递锁给下一个等待者
    arch_mcs_spin_unlock_contended(&next->locked);
}
```

**核心优势：** 每个 CPU 在**本地 cacheline** 上自旋，避免 cache bounce。
- 传统 TATAS (Test-And-Set) 锁：所有 CPU 在同一 cacheline 上自旋，导致 cache line 频繁无效
- MCS 锁：每个等待者在各自的 `node->locked` 上自旋，零竞争

## 五、关键文件索引

| 文件 | 作用 |
|------|------|
| `include/linux/spinlock.h` | 主头文件，API 层，定义 `spin_lock()` 等宏 |
| `include/linux/spinlock_types_raw.h` | `raw_spinlock_t` 定义 |
| `include/linux/spinlock_api_smp.h` | `_raw_spin_*` 内联实现 |
| `include/asm-generic/qspinlock_types.h` | `qspinlock` 位域定义 |
| `include/asm-generic/qspinlock.h` | `arch_spin_*` → `queued_spin_*` 映射 |
| `arch/x86/include/asm/qspinlock.h` | x86 特定优化 (paravirt, virt_spin_lock) |
| `kernel/locking/qspinlock.c` | **核心实现** — 慢速路径 |
| `kernel/locking/mcs_spinlock.h` | MCS 锁基础原语 |
| `kernel/locking/spinlock.c` | `_raw_spin_lock` 函数体 (BUILD_LOCK_OPS) |
| `kernel/locking/spinlock_debug.c` | `CONFIG_DEBUG_SPINLOCK` 调试 |

## 六、API 家族

### 基本 API

```c
spin_lock(lock)              /* 普通加锁，禁用抢占 */
spin_lock_irq(lock)          /* 加锁 + 关中断 */
spin_lock_irqsave(lock,flags)/* 加锁 + 保存并关中断 */
spin_lock_bh(lock)           /* 加锁 + 关 bottom half */
spin_trylock(lock)           /* 尝试获取，不阻塞 */
spin_unlock(lock)            /* 解锁 */
spin_unlock_irq(lock)        /* 解锁 + 开中断 */
spin_unlock_irqrestore(lock, flags)  /* 解锁 + 恢复中断状态 */
spin_unlock_bh(lock)         /* 解锁 + 开 bottom half */
spin_is_locked(lock)         /* 检查锁状态 */
spin_is_contended(lock)      /* 检查是否有竞争者 */
```

### raw_spinlock API (不能被子类化为 RT mutex)

```c
raw_spin_lock(lock)          /* 原始加锁，总是使用 spinlock 实现 */
raw_spin_lock_irqsave(lock, flags)
raw_spin_trylock(lock)
raw_spin_unlock(lock)
/* ... 等等 */
```

### 每种 API 的内部流程

```
spin_lock(lock)
  ↓
raw_spin_lock(&lock->rlock)     ← 映射到 raw 操作
  ↓
_raw_spin_lock(lock)            ← SMP 封装 (禁用抢占 + lockdep)
  ↓
do_raw_spin_lock(lock)          ← 调用 arch_spin_lock
  ↓
arch_spin_lock(&lock->raw_lock) ← 架构特定实现 (queued_spin_lock)
```

## 七、与 PREEMPT_RT 的关系

- 在非 RT 内核 (`CONFIG_PREEMPT_RT=n`)：`spinlock_t` 映射到 `raw_spinlock_t`，行为相同
- 在 RT 内核 (`CONFIG_PREEMPT_RT=y`)：`spinlock_t` 变为基于 rtmutex 的可睡眠锁，`raw_spinlock_t` 保持真正的自旋语义
- 因此中断上下文必须使用 `raw_spinlock_t`

## 八、架构差异

### x86

- 使用 qspinlock (MCS 队列锁)
- 快速路径：`lock cmpxchg` 指令
- Pending 优化：`lock btsl` 指令 (queued_fetch_set_pending_acquire)
- 支持 paravirt (KVM/Xen 虚拟机优化)

### ARM64

- 同样使用 qspinlock
- 快速路径：`ldxr/stxr` (LL/SC) 循环
- 等待循环：使用 WFE/WFI (Wait For Event) 降低功耗
- 混合大小原子操作需要特殊处理

### UP (单处理器)

- 没有竞争，spinlock 是空操作
- 仅保留 lockdep 和调试功能
