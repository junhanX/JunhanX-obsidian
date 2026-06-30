# Linux Kernel Multi-Generational LRU (mglru) 源码深度分析

**内核版本**: Linux 6.18.34  
**分析时间**: 2026-06-02  
**核心文件**: `mm/vmscan.c`, `include/linux/mmzone.h`, `include/linux/mm_inline.h`

---

## 总览

mglru 是 Linux 内核页面回收的全新实现，替代了传统的 active/inactive 双链表 LRU。它通过**多世代**（Generations）+ **多层级**（Tiers）的设计，解决了传统 LRU 的两个核心问题：

1. **扫描效率**：传统 LRU 需要遍历整个 inactive 链表找可回收页面；mglru 直接定位最冷世代
2. **冷热识别精度**：传统 LRU 只有 hot/cold 两级；mglru 用 4 世代 × 4 层级提供更细的分类

### 核心架构图

```
┌─────────────────────────────────────────────────────┐
│                    lru_gen_folio                    │
│                                                     │
│  max_seq ──►  youngest generation number (monotonic)│
│  min_seq[ANON] ──► oldest anon generation           │
│  min_seq[FILE] ──► oldest file generation           │
│                                                     │
│  folios[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES]   │
│    │                                                  │
│    ├─ gen0 (oldest) ──► coldest pages               │
│    ├─ gen1                                            │
│    ├─ gen2                                            │
│    └─ gen3 (newest) ──► hottest pages               │
│                                                     │
│  滑动窗口: [min_seq, max_seq]                        │
│  大小范围:   [MIN_NR_GENS=2, MAX_NR_GENS=4]          │
└─────────────────────────────────────────────────────┘
```

---

## 阶段 1：数据结构全景图

### 1.1 `struct lru_gen_folio` — 核心容器

文件：`include/linux/mmzone.h:490`

```c
struct lru_gen_folio {
    /* aging 推进：递增 youngest generation */
    unsigned long max_seq;
    /* eviction 推进：递增 oldest generation（anon 和 file 分开） */
    unsigned long min_seq[ANON_AND_FILE];  // ANON_AND_FILE = 2
    /* 每个世代的出生时间（jiffies），用于计算 age */
    unsigned long timestamps[MAX_NR_GENS];

    /* 多维 LRU 列表：[世代][anon/file][内存zone] */
    struct list_head folios[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];

    /* 各世代页面数（最终一致性，可能暂时为负） */
    long nr_pages[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];

    /* 指数移动平均：refaulted 和 evicted+protected */
    unsigned long avg_refaulted[ANON_AND_FILE][MAX_NR_TIERS];
    unsigned long avg_total[ANON_AND_FILE][MAX_NR_TIERS];
    unsigned long protected[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];
    atomic_long_t evicted[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];
    atomic_long_t refaulted[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];

    bool enabled;       // mglru 是否启用
    u8 gen;             // 所属 memcg generation
    u8 seg;             // 所属链表段（head/tail/default）
    struct hlist_nulls_node list;  // 全局 reclaim 的 per-node 链表
};
```

**关键设计解读**：

- **三维数组 `folios[gen][type][zone]`**：每个维度都有明确意义
  - `gen`（4个）：时间维度，越老的世代页面越冷
  - `type`（2个）：anon 和 file 分开，因为它们的回收代价不同（anon 需要 swap）
  - `zone`（MAX_NR_ZONES）：按内存 zone 分类，保证 zone 级别的 reclaim 正确性

- **`min_seq` 分 anon/file**：允许 anon 和 file 的滑动窗口不同步，最多相差 `MAX_NR_GENS - MIN_NR_GENS - 1 = 1` 个世代。这在极端 swappiness 场景下很有用——比如 swappiness=0 时只回收 file，file 的 min_seq 可以推进而 anon 不动。

- **`nr_pages` 是 eventually consistent**：在 `reset_batch_size()` 期间可能暂时为负，这是为了批量处理减少锁竞争的设计。

### 1.2 `lruvec` 中的 mglru 部分

文件：`include/linux/mmzone.h:686`

```c
struct lruvec {
    /* 传统 LRU 列表（mglru 启用时不再使用） */
    struct list_head        lists[NR_LRU_LISTS];

#ifdef CONFIG_LRU_GEN
    struct lru_gen_folio    lrugen;          // mglru 核心数据
#ifdef CONFIG_LRU_GEN_WALKS_MMU
    struct lru_gen_mm_state mm_state;        // Bloom filter + MM 状态
#endif
#endif
    struct pglist_data      *pgdat;
};
```

每个 node 有一个 `__lruvec`，每个 memcg 在每个 node 上也有一个 lruvec。

### 1.3 `folio->flags` 中的位域布局

文件：`include/linux/mmzone.h:1147-1154` 和 `kernel/bounds.c:25-30`

```
Page flags 布局（从高到低）:
| SECTION | NODE | ZONE | LAST_CPUPID | KASAN_TAG | LRU_GEN | LRU_REFS | ... | FLAGS |

LRU_GEN_WIDTH = order_base_2(MAX_NR_GENS + 1) = order_base_2(5) = 3 bits
LRU_REFS_WIDTH = MAX_NR_TIERS - 2 = 4 - 2 = 2 bits
```

**`folio->flags` 中 mglru 使用的位**：

| 位域 | 宽度 | 含义 |
|------|------|------|
| `LRU_GEN_MASK` | 3 bits | 存储 `gen + 1`（0 表示不在 LRU 上）|
| `LRU_REFS_MASK` | 2 bits | 额外引用计数（在 PG_referenced 之后）|
| `PG_referenced` | 1 bit | 首次被访问标记 |
| `PG_workingset` | 1 bit | 最热层级标记 |
| `PG_active` | 1 bit | 被隔离时设置 |

**Generation 存储值的含义**：
- `folio->flags` 中的 LRU_GEN 存的是 `gen + 1`（不是直接存 gen）
- `gen = (flags & LRU_GEN_MASK) >> LRU_GEN_PGOFF - 1`
- `0` 表示 folio 不在任何 generation 上
- `1` = gen0（最老）, `2` = gen1, `3` = gen2, `4` = gen3（最年轻）

这样设计的好处：0 天然表示"不在 LRU 上"，不需要额外判断。

### 1.4 `max_seq` / `min_seq` 的滑动窗口

```
时间方向（从左到右，越来越新）:

min_seq ───────────────────────────── max_seq
  │        │        │        │        │
  gen0     gen1     gen2     gen3    (新)
  (最冷)                           (最热)
  └── 滑动窗口 ──┘

窗口大小 = max_seq - min_seq + 1
  最小值: MIN_NR_GENS (2)
  最大值: MAX_NR_GENS (4)
```

**关键规则**：
- `max_seq` 单调递增，在 aging 路径推进
- `min_seq` 单调递增，在 eviction 路径推进
- anon 和 file 的 `min_seq` 独立推进，但 `max_seq` 共享
- 窗口大小必须保持在 `[MIN_NR_GENS, MAX_NR_GENS]` 范围内

---

## 阶段 2：Generation 和 Tier 的概念

### 2.1 为什么 MAX_NR_GENS = 4？

文件：`include/linux/mmzone.h:393-396`

```c
/*
 * MAX_NR_GENS is set to 4 so that the multi-gen LRU can support twice the
 * number of categories of the active/inactive LRU when keeping track of
 * accesses through page tables.
 */
#define MAX_NR_GENS		4U
```

传统 LRU 有 2 个类别：active 和 inactive。mglru 用 4 个世代提供 **2 倍的分类精度**。更重要的是，4 个世代完美支持"第二次机会"机制。

### 2.2 MIN_NR_GENS = 2 与"第二次机会"

文件：`include/linux/mmzone.h:379-386`

```c
/*
 * After a folio is faulted in, the aging needs to check the accessed bit at
 * least twice before handing this folio over to the eviction. The first check
 * clears the accessed bit from the initial fault; the second check makes sure
 * this folio hasn't been used since then. This process, AKA second chance,
 * requires a minimum of two generations, hence MIN_NR_GENS.
 */
#define MIN_NR_GENS		2U
```

**第二次机会的工作流程**：

```
页面 fault 进来
    │
    ▼
进入最新世代 (gen3 = max_seq)
    │
    ▼
第一次 Aging 扫描页表
    │
    ▼
PG_accessed 位 = 1? ──yes──► 清除 accessed 位，页面留在 gen3
    │                              (说明页面自 fault 后被访问过，值得保留)
    no
    ▼
推进到 gen2 (第二次机会)
    │
    ▼
第二次 Aging 扫描页表
    │
    ▼
PG_accessed 位 = 1? ──yes──► 清除 accessed 位，设置 PG_referenced，
    │                            提升到最新世代 (promote)
    no
    ▼
推进到 gen1 (交给 eviction 处理)
```

**为什么需要 2 个最小世代**？因为第一次检查清除初始 fault 的 accessed 位，第二次检查确认页面是否真的被使用过。少于 2 个世代就无法完成这个"第二次机会"机制。

### 2.3 lru_gen_is_active() — 与旧 ABI 的兼容

文件：`include/linux/mm_inline.h:164`

```c
static inline bool lru_gen_is_active(const struct lruvec *lruvec, int gen)
{
    unsigned long max_seq = lruvec->lrugen.max_seq;

    /* 最年轻的两代 = active，其余 = inactive */
    return gen == lru_gen_from_seq(max_seq) ||
           gen == lru_gen_from_seq(max_seq - 1);
}
```

这是 mglru 与传统 LRU ABI 兼容的关键。`/proc/vmstat` 等用户空间接口仍然期望 active/inactive 分类。在 mglru 中：

- **最新两代**（gen2 + gen3，即 max_seq 和 max_seq-1）被视为 **active**
- **最老两代**（gen0 + gen1，即 min_seq 附近）被视为 **inactive**

当窗口大小 = 4 时：
```
gen0 ── inactive (cold)
gen1 ── inactive (warm)
gen2 ── active   (warm)
gen3 ── active   (hot)
```

当窗口大小 = 2 时：
```
gen0 ── inactive
gen1 ── active
```

### 2.4 Tier 系统 — 无锁的热度追踪

文件：`include/linux/mmzone.h:401-421`

```c
/*
 * Each generation is divided into multiple tiers. A folio accessed N times
 * through file descriptors is in tier order_base_2(N).
 *
 * MAX_NR_TIERS is set to 4 so that the multi-gen LRU can support twice the
 * number of categories of the active/inactive LRU.
 */
#define MAX_NR_TIERS		4U
```

**Tier 与 Generation 的区别**：

| | Generation | Tier |
|---|---|---|
| **变更方式** | 需要 LRU 锁（移动链表） | 原子操作 `folio->flags`，无锁 |
| **触发路径** | Aging（页表遍历） | 文件描述符访问路径（buffered access）|
| **变化频率** | 慢（世代推进时） | 快（每次访问时）|
| **维度** | 时间（新旧） | 频率（冷热） |

**4 个 Tier 的分布**：

```
Tier 0: N=0,1 次访问 → PG_referenced 标记
Tier 1: N=2,3 次访问 → LRU_REFS_MASK 额外位
Tier 2: N=4,7 次访问 → LRU_REFS_MASK 更多位
Tier 3: N=8+ 次访问 → PG_workingset 标记（最热）
```

`lru_tier_from_refs()` 的实现：
```c
static inline int lru_tier_from_refs(int refs, bool workingset)
{
    return workingset ? MAX_NR_TIERS - 1 : order_base_2(refs);
}
```

**关键优势**：Tier 的变更不需要获取 LRU 锁，只需要原子修改 `folio->flags`。在 buffered I/O 的访问路径上，这是零锁开销的。Eviction 路径通过比较 `refaulted/(evicted+protected)` 从 tier 0 和 tier 1+ 的数据，判断文件访问模式是否真的热。

### 2.5 PG_referenced / PG_workingset / LRU_REFS_MASK 的协同

文件：`include/linux/mmzone.h:428-449`

```c
#define LRU_REFS_FLAGS		(LRU_REFS_MASK | BIT(PG_referenced))
```

两套访问路径使用不同的标记策略：

**路径 1：页表访问**（Mapped pages，通过 page table）
- 第一次：清除 accessed 位，设置 `PG_referenced`
- 第二次：设置 `PG_workingset`，提升到最新世代
- `LRU_REFS_MASK` 不使用

**路径 2：文件描述符访问**（File descriptor，通过 buffered I/O）
- `PG_referenced` → `LRU_REFS_MASK` 额外位 → `PG_workingset`
- 全部位满后，eviction 路径**懒提升**到第二老世代

---

## 阶段 3：Aging 机制（页表遍历）

### 3.1 整体架构

Aging 是推进 `max_seq` 的过程。它遍历所有进程的页表，清除 accessed 位，将被访问过的页面提升到新世代。

**入口函数**：`try_to_inc_max_seq()` → `walk_mm()`

```
try_to_inc_max_seq()
    │
    ├── should_walk_mmu()? ──yes──► 页表遍历路径
    │                                  │
    │                                  ▼
    │                           set_mm_walk() 创建 walk 结构
    │                                  │
    │                                  ▼
    │                           walk_mm() → walk_page_range()
    │                                  │
    │                                  ▼
    │                           walk_pud_range() → walk_update_folio()
    │
    └── no ──► iterate_mm_list_nowalk()（备用路径）
                │
                ▼
         inc_max_seq() 推进 max_seq
```

### 3.2 try_to_inc_max_seq() — Aging 入口

文件：`mm/vmscan.c:4059-4106`

```c
static bool try_to_inc_max_seq(struct lruvec *lruvec, unsigned long seq,
			       int swappiness, bool force_scan)
{
    struct lru_gen_mm_walk *walk;
    struct lru_gen_mm_state *mm_state = get_mm_state(lruvec);

    /* 如果没有 mm_state（无 LRU_GEN_WALKS_MMU），直接推进 */
    if (!mm_state)
        return inc_max_seq(lruvec, seq, swappiness);

    /* 如果 mm_state 已经跟上了，不需要推进 */
    if (seq <= READ_ONCE(mm_state->seq))
        return false;

    /* 硬件不支持 accessed bit → 走备用路径 */
    if (!should_walk_mmu()) {
        success = iterate_mm_list_nowalk(lruvec, seq);
        goto done;
    }

    /* 主路径：创建 walk 结构，遍历页表 */
    walk = set_mm_walk(NULL, true);
    if (!walk) {
        success = iterate_mm_list_nowalk(lruvec, seq);
        goto done;
    }

    walk->lruvec = lruvec;
    walk->seq = seq;
    walk->swappiness = swappiness;
    walk->force_scan = force_scan;

    /* 遍历所有 mm_struct */
    while ((mm = get_next_mm(walk))) {
        walk_mm(mm, walk);
        /* ... */
    }

    /* 推进 max_seq */
    success = inc_max_seq(lruvec, seq, swappiness);
    /* ... */
}
```

**设计解读**：
1. `get_next_mm()` 从全局 `mm_struct` 链表中取下一个 mm
2. 每个 mm 用 `walk_page_range()` 遍历其页表
3. Bloom filter 优化：跳过不需要遍历的 PMD
4. 遍历完成后调用 `inc_max_seq()` 推进世代

### 3.3 walk_mm() — 遍历单个 mm

文件：`mm/vmscan.c:3814`

```c
static void walk_mm(struct mm_struct *mm, struct lru_gen_mm_walk *walk)
{
    static const struct mm_walk_ops mm_walk_ops = {
        .test_walk = should_skip_vma,    // 跳过特殊 VMA
        .p4d_entry = walk_pud_range,     // 从 P4D 层级开始
        .walk_lock = PGWALK_RDLOCK,      // 读锁
    };

    do {
        DEFINE_MAX_SEQ(lruvec);

        /* 另一个线程可能已经推进了 max_seq */
        if (walk->seq != max_seq)
            break;

        if (mmap_read_trylock(mm)) {
            err = walk_page_range(mm, walk->next_addr, ULONG_MAX,
                                  &mm_walk_ops, walk);
            mmap_read_unlock(mm);
        }

        /* 批量处理页面更新 */
        if (walk->batched) {
            spin_lock_irq(&lruvec->lru_lock);
            reset_batch_size(walk);
            spin_unlock_irq(&lruvec->lru_lock);
        }

        cond_resched();
    } while (err == -EAGAIN);
}
```

**批量处理设计**：
- `walk->nr_pages[gen][type][zone]` 批量记录页面变化
- `walk->batched` 达到阈值后，一次性在 LRU 锁下批量移动
- 这减少了锁的获取次数，提高了并发性能

### 3.4 walk_update_folio() — 更新单个 folio 的世代

文件：`mm/vmscan.c:3508`

```c
/* 当页表遍历发现一个 young PTE 时调用 */
static void walk_update_folio(struct lru_gen_mm_walk *walk,
                              struct folio *folio, ...)
{
    /* 如果 folio 已经被访问过（young），更新其世代 */
    if (young) {
        /* 更新到最新世代 */
        new_gen = lru_gen_from_seq(walk->seq);
        update_batch_size(walk, folio, old_gen, new_gen);
    }
}
```

### 3.5 lru_gen_look_around() — 空间局部性优化

文件：`mm/vmscan.c:4239`

这是 mglru 中一个非常精巧的优化。当 eviction 路径在 rmap 遍历中发现一个 young PTE 时，它**顺便扫描相邻的 PTE**：

```c
bool lru_gen_look_around(struct page_vma_mapped_walk *pvmw)
{
    /* 清除当前 PTE 的 young bit */
    if (!ptep_clear_young_notify(vma, addr, pte))
        return false;

    /* 扫描相邻 PTE（在一个 PMD 范围内）*/
    start = max(addr & PMD_MASK, vma->vm_start);
    end = min(addr | ~PMD_MASK, vma->vm_end - 1) + 1;

    for (addr = start; addr != end; addr += PAGE_SIZE) {
        pte_t ptent = ptep_get(pte + i);
        if (ptep_clear_young_notify(vma, addr, pte + i))
            young++;  // 计数发现的 young PTE
    }

    /* 反馈给页表遍历：如果扫描效率高，加入 Bloom filter */
    if (mm_state && suitable_to_scan(i, young))
        update_bloom_filter(mm_state, max_seq, pvmw->pmd);
}
```

**这个函数的巧妙之处**：
1. 利用空间局部性：如果一个页面被访问了，相邻页面也很可能被访问
2. 在持有 PTL（页表锁）时顺便做，减少额外锁开销
3. 将扫描结果反馈给 Bloom filter，形成 **eviction ↔ aging 的闭环**

### 3.6 inc_max_seq() — 推进 max_seq

文件：`mm/vmscan.c:3994`

```c
static bool inc_max_seq(struct lruvec *lruvec, unsigned long seq, int swappiness)
{
    /* 确保当前 max_seq 还没被别人推进 */
    if (seq < READ_ONCE(lrugen->max_seq))
        return false;

    spin_lock_irq(&lruvec->lru_lock);

    /* 如果所有类型的世代数都是 MAX_NR_GENS，先推进 min_seq */
    for (type = 0; type < ANON_AND_FILE; type++) {
        if (get_nr_gens(lruvec, type) != MAX_NR_GENS)
            continue;
        if (inc_min_seq(lruvec, type, swappiness))
            continue;
        /* 如果 inc_min_seq 失败，释放锁重试 */
        spin_unlock_irq(&lruvec->lru_lock);
        cond_resched();
        goto restart;
    }

    /* 更新 active/inactive 的兼容计数 */
    prev = lru_gen_from_seq(lrugen->max_seq - 1);
    next = lru_gen_from_seq(lrugen->max_seq + 1);
    for (type = 0; type < ANON_AND_FILE; type++)
        for (zone = 0; zone < MAX_NR_ZONES; zone++) {
            long delta = lrugen->nr_pages[prev][type][zone] -
                         lrugen->nr_pages[next][type][zone];
            __update_lru_size(lruvec, lru, zone, delta);
            __update_lru_size(lruvec, lru + LRU_ACTIVE, zone, -delta);
        }

    /* 记录新世代的出生时间 */
    WRITE_ONCE(lrugen->timestamps[next], jiffies);

    /* 原子地推进 max_seq */
    smp_store_release(&lrugen->max_seq, lrugen->max_seq + 1);
    spin_unlock_irq(&lruvec->lru_lock);
}
```

---

## 阶段 4：Bloom Filter 优化

### 4.1 为什么需要 Bloom Filter？

文件：`mm/vmscan.c:2807-2828`

页表遍历的成本是 O(所有页面的 PTE 数量)。对于大内存系统，这可能涉及数百万个 PTE。Bloom filter 用于**减少无效的页表遍历**。

```c
/*
 * Bloom filters with m=1<<15, k=2 and the false positive rates of
 * ~1/5 when n=10,000 and ~1/2 when n=20,000.
 *
 * Page table walkers use one of the two filters to reduce their search space.
 * To get rid of non-leaf entries that no longer have enough leaf entries,
 * the aging uses the double-buffering technique to flip to the other filter
 * each time it produces a new generation.
 */
#define BLOOM_FILTER_SHIFT	15
```

**参数**：
- `m = 2^15 = 32768` bits = 4KB
- `k = 2` 个 hash 函数
- 插入 10,000 项时误判率 ~1/5
- 插入 20,000 项时误判率 ~1/2

### 4.2 双缓冲设计

```c
struct lru_gen_mm_state {
    unsigned long seq;              // 同步于 max_seq
    struct list_head *head;         // 当前迭代继续的位置
    struct list_head *tail;         // 上次迭代结束的位置
    unsigned long *filters[NR_BLOOM_FILTERS];  // 双缓冲
    unsigned long stats[NR_HIST_GENS][NR_MM_STATS];
};

#define NR_BLOOM_FILTERS	2
```

**为什么需要两个 filter**？

```
迭代 N: 使用 filter[0] 记录 young PMD
         页表遍历只查询 filter[0]

迭代 N+1: filter[0] 已经满了（大量 PMD 已插入）
          翻转到 filter[1]，清空后重新使用
          页表遍历查询 filter[1]

关键点：old filter 中的 non-leaf entry 如果已经没有
        足够的 leaf entry，就可以被淘汰掉
```

### 4.3 核心操作

```c
/* 查询：PMD 是否可能在 filter 中？*/
static bool test_bloom_filter(struct lru_gen_mm_state *mm_state,
                              unsigned long seq, void *item)
{
    int key[2];
    unsigned long *filter;
    int gen = filter_gen_from_seq(seq);  // seq % 2

    filter = READ_ONCE(mm_state->filters[gen]);
    if (!filter)
        return true;  // filter 不存在 → 全部扫描

    get_item_key(item, key);  // Knuth hash → 2 个 key
    return test_bit(key[0], filter) && test_bit(key[1], filter);
}

/* 插入：记录一个 young PMD */
static void update_bloom_filter(...) { ... }

/* 重置：翻转 filter */
static void reset_bloom_filter(...) { ... }
```

### 4.4 反馈循环

```
Eviction 路径 (rmap walk)
    │
    │ lru_gen_look_around()
    │ 发现大量 young PTE
    │
    ▼
update_bloom_filter(mm_state, max_seq, pmd)
    │
    ▼
下次 Aging (page table walk)
    │
    │ test_bloom_filter()
    │ PMD 在 filter 中 → 遍历其 PTE
    │ PMD 不在 filter 中 → 跳过
    │
    ▼
再次更新 Bloom filter（闭环）
```

---

## 阶段 5：Eviction 机制（回收路径）

### 5.1 整体调用链

```
shrink_lruvec()
    │
    ├── lru_gen_enabled()? ──yes──► lru_gen_shrink_lruvec()
    │                                 │
    │                                 ├── lru_add_drain()
    │                                 ├── set_mm_walk()
    │                                 ├── try_to_shrink_lruvec()
    │                                 │     │
    │                                 │     ├── get_nr_to_scan()
    │                                 │     │     └── should_run_aging()
    │                                 │     │           └── try_to_inc_max_seq()
    │                                 │     │
    │                                 │     └── evict_folios()
    │                                 │           ├── isolate_folios()
    │                                 │           ├── try_to_inc_min_seq()
    │                                 │           └── shrink_folio_list()
    │                                 │
    │                                 └── lru_gen_rotate_memcg()
    │
    └── no (传统路径) ──► get_scan_count() → shrink_list()
```

### 5.2 lru_gen_shrink_lruvec() — mglru 回收入口

文件：`mm/vmscan.c:5054`

```c
static void lru_gen_shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)
{
    lru_add_drain();           // 把 LRU 缓存中的页面刷入主 LRU
    blk_start_plug(&plug);     // 批量 block IO

    set_mm_walk(NULL, sc->proactive);

    if (try_to_shrink_lruvec(lruvec, sc))
        lru_gen_rotate_memcg(lruvec, MEMCG_LRU_YOUNG);  // 回收成功，提升 memcg

    clear_mm_walk();
    blk_finish_plug(&plug);
}
```

### 5.3 try_to_shrink_lruvec() — 回收循环

文件：`mm/vmscan.c:4905`

```c
static bool try_to_shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)
{
    int swappiness = get_swappiness(lruvec, sc);

    while (true) {
        nr_to_scan = get_nr_to_scan(lruvec, sc, swappiness);
        if (nr_to_scan <= 0)
            break;

        delta = evict_folios(nr_to_scan, lruvec, sc, swappiness);
        if (!delta)
            break;

        scanned += delta;
        if (scanned >= nr_to_scan)
            break;

        if (should_abort_scan(lruvec, sc))
            break;

        cond_resched();
    }

    /* 如果最老世代的 file 页面大量因 dirty 无法回收，唤醒 flusher */
    if (sc->nr.unqueued_dirty && sc->nr.unqueued_dirty == sc->nr.file_taken)
        wakeup_flusher_threads(WB_REASON_VMSCAN);

    return nr_to_scan < 0;  // < 0 表示需要 rotate memcg
}
```

### 5.4 get_nr_to_scan() — 计算扫描数量 + 触发 Aging

文件：`mm/vmscan.c:4848`

```c
static long get_nr_to_scan(struct lruvec *lruvec, struct scan_control *sc, int swappiness)
{
    unsigned long nr_to_scan;

    /* 检查是否需要先执行 Aging */
    if (should_run_aging(lruvec, max_seq, swappiness, &nr_to_scan)) {
        try_to_inc_max_seq(lruvec, max_seq, swappiness, false);
        return 0;  // 先 aging，下次再 scan
    }

    /* 计算从最老世代可以扫描的页面数 */
    /* ... */
    return nr_to_scan;
}
```

### 5.5 should_run_aging() — 何时需要 Aging

文件：`mm/vmscan.c:4814`

```c
static bool should_run_aging(struct lruvec *lruvec, unsigned long max_seq,
			     int swappiness, unsigned long *nr_to_scan)
{
    /* 必须执行 aging：eviction 已经不可能了（窗口满了）*/
    if (evictable_min_seq(min_seq, swappiness) + MIN_NR_GENS > max_seq)
        return true;

    /* 统计所有世代的页面总数 */
    for_each_evictable_type(type, swappiness)
        for (seq = min_seq[type]; seq <= max_seq; seq++)
            for (zone = 0; zone < MAX_NR_ZONES; zone++)
                size += lrugen->nr_pages[gen][type][zone];

    *nr_to_scan = size;

    /* 刚好到达边界：最好也执行 aging */
    return evictable_min_seq(min_seq, swappiness) + MIN_NR_GENS == max_seq;
}
```

**触发 Aging 的两个条件**：
1. `min_seq + 2 > max_seq`（窗口收缩到最小，不能再 eviction，必须 aging）
2. `min_seq + 2 == max_seq`（刚好在边界，提前 aging 避免后续无法回收）

### 5.6 evict_folios() — 核心回收函数

文件：`mm/vmscan.c:4723`

```c
static int evict_folios(unsigned long nr_to_scan, struct lruvec *lruvec,
			struct scan_control *sc, int swappiness)
{
    LIST_HEAD(list);   // 隔离出来的 folio 链表
    LIST_HEAD(clean);  // 需要重试的 clean folio 链表

    spin_lock_irq(&lruvec->lru_lock);

    /* 1. 从最老世代隔离 folio */
    scanned = isolate_folios(nr_to_scan, lruvec, sc, swappiness, &type, &list);

    /* 2. 尝试推进 min_seq（回收完的老世代可以丢弃）*/
    scanned += try_to_inc_min_seq(lruvec, swappiness);

    /* 如果隔离失败或没有页面，返回 */
    if (list_empty(&list))
        return scanned;

    spin_unlock_irq(&lruvec->lru_lock);

retry:
    /* 3. 实际回收：swap out, writeback, free */
    reclaimed = shrink_folio_list(&list, pgdat, sc, &stat, false, memcg);

    /* 4. 处理未能回收的 folio */
    list_for_each_entry_safe_reverse(folio, next, &list, lru) {
        if (!folio_evictable(folio)) {
            list_del(&folio->lru);
            folio_putback_lru(folio);  // 不可回收，放回
            continue;
        }

        /* clean folio重试一次 */
        if (!skip_retry && !folio_test_active(folio) &&
            !folio_mapped(folio) && !folio_test_dirty(folio) &&
            !folio_test_writeback(folio)) {
            list_move(&folio->lru, &clean);
            continue;
        }

        /* 被拒绝的 folio 放到最老世代（标记 PG_active）*/
        if (lru_gen_folio_seq(lruvec, folio, false) == min_seq[type])
            set_mask_bits(&folio->flags.f, LRU_REFS_FLAGS, BIT(PG_active));
    }

    /* 5. 把未回收的放回 LRU，更新统计 */
    spin_lock_irq(&lruvec->lru_lock);
    move_folios_to_lru(lruvec, &list);
    /* ... 更新 vmstat ... */
    spin_unlock_irq(&lruvec->lru_lock);

    /* 6. 重试 clean folio */
    list_splice_init(&clean, &list);
    if (!list_empty(&list)) {
        skip_retry = true;
        goto retry;
    }

    return scanned;
}
```

### 5.7 try_to_inc_min_seq() — 推进最老世代

文件：`mm/vmscan.c:3937`

```c
static bool try_to_inc_min_seq(struct lruvec *lruvec, int swappiness)
{
    /* 找到最老的有页面的世代 */
    for_each_evictable_type(type, swappiness) {
        while (min_seq[type] + MIN_NR_GENS <= lrugen->max_seq) {
            gen = lru_gen_from_seq(min_seq[type]);

            for (zone = 0; zone < MAX_NR_ZONES; zone++)
                if (!list_empty(&lrugen->folios[gen][type][zone]))
                    goto next;  // 还有页面，不能跳过

            min_seq[type]++;  // 空世代，直接跳过
        }
    next:
        ;
    }

    /* 防止 anon/file min_seq 差距过大 */
    if (swappiness && swappiness <= MAX_SWAPPINESS) {
        unsigned long seq = lrugen->max_seq - MIN_NR_GENS;
        if (min_seq[LRU_GEN_ANON] > seq && min_seq[LRU_GEN_FILE] < seq)
            min_seq[LRU_GEN_ANON] = seq;
        else if (min_seq[LRU_GEN_FILE] > seq && min_seq[LRU_GEN_ANON] < seq)
            min_seq[LRU_GEN_FILE] = seq;
    }

    /* 原子地更新 min_seq */
    for_each_evictable_type(type, swappiness)
        WRITE_ONCE(lrugen->min_seq[type], min_seq[type]);

    return success;
}
```

### 5.8 inc_min_seq() — 推进并处理残留 folio

文件：`mm/vmscan.c:3884`

```c
static bool inc_min_seq(struct lruvec *lruvec, int type, int swappiness)
{
    int remaining = MAX_LRU_BATCH;
    int old_gen = lru_gen_from_seq(lrugen->min_seq[type]);

    /* 遍历最老世代的所有 folio */
    for (zone = 0; zone < MAX_NR_ZONES; zone++) {
        while (!list_empty(head)) {
            folio = lru_to_folio(head);
            int refs = folio_lru_refs(folio);
            bool workingset = folio_test_workingset(folio);

            /* 提升 folio 到下一代（因为 min_seq 要前进了）*/
            new_gen = folio_inc_gen(lruvec, folio, false);
            list_move_tail(&folio->lru,
                          &lrugen->folios[new_gen][type][zone]);

            /* 更新 protected 计数（用于 workingset 检测）*/
            if (refs + workingset != BIT(LRU_REFS_WIDTH) + 1) {
                int tier = lru_tier_from_refs(refs, workingset);
                WRITE_ONCE(lrugen->protected[hist][type][tier],
                          lrugen->protected[hist][type][tier] + delta);
            }

            if (!--remaining)
                return false;  // 批量限制，下次继续
        }
    }

    WRITE_ONCE(lrugen->min_seq[type], lrugen->min_seq[type] + 1);
    return true;
}
```

**关键点**：当 `min_seq` 要前进时，最老世代中的 folio 不能直接丢弃——它们被**提升**到下一代（`folio_inc_gen()`），同时记录到 `protected` 计数中用于 workingset 分析。

---

## 阶段 6：Memcg LRU（可扩展性设计）

### 6.1 传统 Memcg 的问题

传统 reclaim 用 `mem_cgroup_iter()` 遍历所有 memcg，这是 O(n) 操作。当系统有成千上万个 memcg 时，这会变成性能瓶颈。

### 6.2 Memcg 分代设计

文件：`include/linux/mmzone.h:561-603`

```c
/*
 * For each node, memcgs are divided into two generations: old and young.
 * For each generation, memcgs are randomly sharded into multiple bins
 * to improve scalability.
 *
 * MEMCG_NR_GENS = 3: 保证 lockless 读取时，旧值(seq-1)不会回绕到 young
 * MEMCG_NR_BINS = 8: 每个分代 8 个 bin 分散 memcg
 */
#define MEMCG_NR_GENS	3
#define MEMCG_NR_BINS	8
```

**结构**：
```
Old Generation (seq)                Young Generation (seq+1)
  ├── bin[0] ── memcg ── memcg       ├── bin[0] ── memcg
  ├── bin[1] ── memcg                ├── bin[1]
  ├── bin[2]                         ├── bin[2] ── memcg ── memcg
  ├── ...                            ├── ...
  └── bin[7]                         └── bin[7] ── memcg

每个 bin 内部分为三段：head ── default ── tail
```

### 6.3 四种操作

```c
enum {
    MEMCG_LRU_NOP,     // 无操作
    MEMCG_LRU_HEAD,    // 移到当前代的随机 bin 的 head
    MEMCG_LRU_TAIL,    // 移到当前代的随机 bin 的 tail
    MEMCG_LRU_OLD,     // 移到 old 代的随机 bin（重置 seg = default）
    MEMCG_LRU_YOUNG,   // 移到 young 代的随机 bin（重置 seg = default）
};
```

**触发事件**：
| 事件 | 操作 |
|------|------|
| 超过 soft limit | MEMCG_LRU_HEAD |
| 首次 reclaim below low | MEMCG_LRU_TAIL |
| reclaim offlined/below threshold | MEMCG_LRU_TAIL |
| 第二次 reclaim offlined | MEMCG_LRU_YOUNG |
| reclaim below min | MEMCG_LRU_YOUNG |
| aging 完成 | MEMCG_LRU_YOUNG |
| memcg offline | MEMCG_LRU_OLD |

### 6.4 lru_gen_rotate_memcg()

文件：`mm/vmscan.c:4348`

```c
static void lru_gen_rotate_memcg(struct lruvec *lruvec, int op)
{
    /* 根据操作类型，将 memcg 移动到适当的位置 */
    /* MEMCG_LRU_YOUNG → 移到年轻代 */
    /* MEMCG_LRU_OLD → 移到老年代 */
    /* MEMCG_LRU_HEAD → 移到当前代的 head */
    /* MEMCG_LRU_TAIL → 移到当前代的 tail */

    /* 当 old 代所有 bin 都为空时，推进 memcg generation counter */
    /* 这保证了所有 memcg 最终都会被公平地扫描到 */
}
```

### 6.5 shrink_many() — 批量回收

文件：`mm/vmscan.c:4984`

```c
static void shrink_many(struct pglist_data *pgdat, struct scan_control *sc)
{
    /* 遍历 old 代的 bin */
    for (bin = first_bin; ...; bin = (bin + 1) % MEMCG_NR_BINS) {
        for_each_memcg_in_bin(memcg, bin) {
            lruvec = mem_cgroup_lruvec(memcg, pgdat);

            op = shrink_one(lruvec, sc);

            if (op == MEMCG_LRU_YOUNG) {
                lru_gen_rotate_memcg(lruvec, MEMCG_LRU_YOUNG);
            } else if (op) {
                lru_gen_rotate_memcg(lruvec, op);
            }

            if (should_abort_scan(lruvec, sc))
                return;
        }
    }
}
```

---

## 阶段 7：Workingset 和 Thrashing 防护

### 7.1 指数移动平均

```c
struct lru_gen_folio {
    unsigned long avg_refaulted[ANON_AND_FILE][MAX_NR_TIERS];
    unsigned long avg_total[ANON_AND_FILE][MAX_NR_TIERS];
};
```

每个 tier 维护一个 EMA（Exponential Moving Average）：
- `avg_refaulted`：从该 tier 回收后 refault 的页面数
- `avg_total`：该 tier 回收的 + 保护的页面总数

**判断标准**：如果 `refaulted/total` 在 tier 0 很高，说明这个 tier 的页面是热的，应该保护。如果 tier 1+ 的比率低，说明这些页面是真正的 cold。

### 7.2 Thrashing 防护：min_ttl_ms

文件：`mm/vmscan.c:4190-4226`

```c
static unsigned long lru_gen_min_ttl __read_mostly;

static void lru_gen_age_node(struct pglist_data *pgdat, struct scan_control *sc)
{
    unsigned long min_ttl = READ_ONCE(lru_gen_min_ttl);
    bool reclaimable = !min_ttl;  // min_ttl=0 表示不防护

    /* 遍历所有 memcg，检查是否有世代 >= min_ttl */
    memcg = mem_cgroup_iter(NULL, NULL, NULL);
    do {
        lruvec = mem_cgroup_lruvec(memcg, pgdat);
        if (!reclaimable)
            reclaimable = lruvec_is_reclaimable(lruvec, sc, min_ttl);
    } while ((memcg = mem_cgroup_iter(NULL, memcg, NULL)));

    /* 如果所有世代都太"年轻"（无法回收），触发 OOM kill */
    if (!reclaimable && mutex_trylock(&oom_lock)) {
        out_of_memory(&oc);
        mutex_unlock(&oom_lock);
    }
}
```

**工作原理**：
1. 用户设置 `min_ttl_ms`（例如 1000 = 1秒）
2. kswapd 检查：最老的世代是否比 `min_ttl_ms` 更老？
3. 如果是 → 正常回收
4. 如果否（所有世代都比 min_ttl 年轻）→ **触发 OOM kill** 而不是 thrashing

**为什么有效**：thrashing 发生在页面被反复回收又反复 fault 回来。通过保护一定时间窗口内的页面不被回收，打破了这个循环。代价是如果内存真的不够，直接 OOM kill 一个进程而不是让整个系统卡死。

---

## 阶段 8：接口与控制面

### 8.1 sysfs 接口

```
/sys/kernel/mm/lru_gen/
├── enabled        # 开关（位掩码）
└── min_ttl_ms     # TTL 防护（毫秒）
```

**enabled 的位定义**：

| 位 | 含义 |
|---|---|
| 0x0001 | 主开关 |
| 0x0002 | 清除 leaf PTE 的 accessed bit（批量）|
| 0x0004 | 清除 non-leaf PTE 的 accessed bit |
| 0x0007 | 全部启用 |

```c
static ssize_t enabled_show(struct kobject *kobj, char *buf)
{
    unsigned int caps = 0;
    if (get_cap(LRU_GEN_CORE))      caps |= BIT(LRU_GEN_CORE);
    if (should_walk_mmu())          caps |= BIT(LRU_GEN_MM_WALK);
    if (should_clear_pmd_young())   caps |= BIT(LRU_GEN_NONLEAF_YOUNG);
    return sysfs_emit(buf, "0x%04x\n", caps);
}
```

### 8.2 debugfs 接口

```
/sys/kernel/debug/lru_gen       # 工作集估算 + 主动回收
/sys/kernel/debug/lru_gen_full  # 额外调试统计（需要 CONFIG_LRU_GEN_STATS）
```

**seq_file 输出格式**：
```
memcg  memcg_id  memcg_path
   node  node_id
       min_gen_nr  age_in_ms  nr_anon_pages  nr_file_pages
       ...
       max_gen_nr  age_in_ms  nr_anon_pages  nr_file_pages
```

**写入命令**：

```
# 创建新世代（工作集估算）
+ memcg_id node_id max_gen_nr [can_swap [force_scan]]

# 主动回收（Proactive reclaim）
- memcg_id node_id min_gen_nr [swappiness [nr_to_reclaim]]
```

---

## 附录：关键数据流

### 页面生命周期

```
Fault in
    │
    ▼
lru_gen_add_folio() ──► 进入最新世代 (gen = max_seq)
    │
    ▼
┌──────────────────────────────────────────────┐
│              Aging 循环                       │
│                                              │
│  页表遍历 → 清除 young bit                   │
│  如果页面被访问 → 提升到最新世代              │
│  如果页面未被访问 → 自然老化（留在当前世代）    │
│  max_seq++                                  │
└──────────────────────────────────────────────┘
    │
    ▼
进入最老世代（min_seq 对应的 gen）
    │
    ▼
┌──────────────────────────────────────────────┐
│             Eviction 循环                     │
│                                              │
│  isolate_folios() → 从最老世代隔离            │
│  shrink_folio_list() → 尝试回收               │
│    ├── 可回收 → swap/writeback/free          │
│    └── 不可回收 → 放回并提升世代              │
│  min_seq++                                  │
└──────────────────────────────────────────────┘
```

### 文件描述符访问路径（无锁 Tier 提升）

```
Buffered read/write
    │
    ▼
mark_page_accessed() / folio_mark_accessed()
    │
    ▼
lru_gen_inc_refs()
    │
    ▼
原子操作修改 folio->flags：
    PG_referenced → LRU_REFS_MASK 位 → PG_workingset
    │
    ▼
Eviction 路径看到 PG_workingset：
    懒提升到第二老世代（不移动链表）
```

---

## 总结

mglru 的核心创新可以概括为三点：

1. **时间维度**（Generations）：用滑动窗口的多世代替代 active/inactive 双链表，使回收时直接定位最冷页面，无需遍历
2. **频率维度**（Tiers）：在每代内通过无锁的 tier 系统区分冷热，在 buffered I/O 路径零开销
3. **可扩展性**（Memcg LRU）：用分代 + bin 打散替代 O(n) 遍历，支持大规模容器场景
