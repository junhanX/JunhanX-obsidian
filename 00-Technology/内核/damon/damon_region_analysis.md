# DAMON 以 Region 为粒度的设计分析

## 1. 为什么以 Region 为粒度，而非 Page 为粒度？

核心原因：**控制监控开销，使其不随监控内存规模线性增长。**

DAMON 的核心设计假设是：

> **相邻的物理页通常具有相似的访问频率。**

基于此，DAMON 将连续的地址空间聚合为一个 **region**，每个 region 内的所有 page 被认为冷热相近。每个 `sampling interval` 只在每个 region 内**随机抽取一个 page** 做采样（检查 PTE Accessed bit），若该页被访问，则认为整个 region 本轮"被访问了"，累加 `nr_accesses`。监控开销由此与 **region 数量** 绑定，而非总内存规模。用户通过 `min_nr_regions` 和 `max_nr_regions` 显式约束开销。

> 📖 *"DAMON groups adjacent pages that assumed to have the same access frequencies into a region... The monitoring overhead is therefore controllable by setting the number of regions."* — DAMON Design Doc

---

## 2. 如何划分 Region？

在每个 `aggregation interval` 内，依次执行 **Merge → Split** 两阶段自适应调整。

### 2.1 初始化

监控启动时将目标地址空间**均匀切为 `min_nr_regions` 个等大小 region**，`nr_accesses` 初始为 0。

### 2.2 Merge 阶段 (`kdamond_merge_regions`)

遍历相邻 region，满足以下条件则合并：

| 条件 | 含义 |
|---|---|
| `|nr_accesses(L) - nr_accesses(R)| < threshold` | 访问频率接近 |
| `size(L) + size(R) ≤ total_size / min_nr_regions` | 合并后不超最大允许 size |

合并后的 `nr_accesses` 为**按 size 加权平均**：

```
nr_accesses_new = (nr_L × sz_L + nr_R × sz_R) / (sz_L + sz_R)
```

**激进合并优化**（commit `310d6c15e910`，2024-07）：若合并后 `nr_regions` 仍超 `max_nr_regions`，则翻倍 threshold 重新合并，循环直到满足上限或 threshold 达到理论最大值 `aggr_interval / sample_interval`。

### 2.3 Split 阶段 (`kdamond_split_regions`)

仅在 `nr_regions ≤ max_nr_regions / 2` 时执行，避免超出上限。每个 region 被切为 **2 或 3** 份：

- **切 2 份**（默认）：切点随机选取在 region 大小的 10% ~ 90% 之间（`damon_rand(1, 10) × sz / 10`）
- **切 3 份**：当 region 数无变化且仍 `< max_nr_regions / 3` 时触发，更激进探测冷热边界

切分点**随机化**是算法精髓——若仅中点切分可能永远无法命中真实冷热边界；随机切分通过概率性探测，随聚合周期收敛至真实边界。

### 2.4 收敛示意

```
初始:  [====== region A ======][====== region B ======]
采样:  A 冷 / B 热 → 不会被 merge
Split: A 随机切出 A1, A2
      A1 冷, A2 冷 → 下轮 merge 合回
      A1 热, A2 冷 → 与 B 形成稳定边界，不再合并
```

### 2.5 关键参数

| 参数 | 作用 |
|---|---|
| `min_nr_regions` | 下限：限制单个 region 最大大小 `total_size / min_nr_regions`，防止过度合并 |
| `max_nr_regions` | 上限：硬约束开销，split 在超过一半时自动停止 |
| `sampling interval` | 单次采样等待窗口（默认 5 ms） |
| `aggregation interval` | 输出完整冷热快照的周期（默认 100 ms） |

---

## 3. 以 Region 冷热判断 Page 冷热是否准确？

**结论：有一定误差，合理调参下足够有效。DAMON 设计者坦承这是 "best-effort quality"。**

### 3.1 不准确的来源

#### A. "相邻页访问频率相似"假设可能不成立

若 10 MiB region 内仅 10 个 4K 页是热点（占 0.04%），每个 sampling interval 只随机采一页，命中概率极低：
- 该 region 的 `nr_accesses` 始终为 0 → **漏报热点**
- Adaptive split 终将切开，但收敛需时，热点可能在此期间迁移

#### B. 收敛速度跟不上访问模式迁移

Workload 快速迁移热点时（如数据库扫描指针），DAMON 刚 split 出正确边界，热点已移走。

#### C. 大 region + 稀疏热点 → 检测能力极差

Intel 的 Aravinda Prasad 在 2024-03 邮件指出：10 GB region 在默认参数下每 aggregation interval 仅采样约 20 个 page，远不够覆盖稀疏访问模式。SeongJae Park 回应承认此局限。

#### D. 默认 aggregation interval 过短

> *"On large systems having limited bandwidth, 100 millisecond aggregation interval could be not long enough."* — SeongJae Park, 2024-11

SK hynix HMSDK 项目实际使用设为 2 秒。

### 3.2 提升准确性的调参建议

| 措施 | 说明 |
|---|---|
| **增大 aggregation interval** | 1~2 秒级，让足够访问事件累积，能区分 hot vs cold |
| **利用 `age` 维度** | `nr_accesses == 0` 但 `age` 很小的 region 可能是刚冷下来的热点（recency） |
| **提高 region 数上限** | `max_nr_regions` 越大，粒度越细，开销越大 |
| **固定 region 数** | `min_nr_regions == max_nr_regions` 关闭自适应，严格均匀切分 |
| **intervals auto-tuning** | 使用 `intervals_goal` 机制让 DAMON 自动调整间隔（推荐 `access_bp = 4%`） |

### 3.3 场景适应性总结

| 场景 | 准确性 |
|---|---|
| **冷内存识别 / proactive reclaim** | ✅ 很好 — 找"一直不访问"的大块冷区域是 DAMON 最强场景 |
| **热点识别 / 热页迁移** | ⚠️ 需仔细调参 — 默认参数易漏热点，需增大 aggr_interval、关注 age |
| **精准到 page 级追踪** | ❌ 不适合 — page 级请用 idle page tracking 或 PEBS |
| **稀疏热点 + 大内存** | ❌ 先天不足 — 采样率太低，收敛时间长 |
| **冷热边界稳定的 workload** | ✅ 很好 — adaptive region adjustment 能有效收敛 |

### 3.4 一句话总结

> DAMON 的 region 粒度是一种**以精度换可控开销**的设计。它不保证每页冷热判断准确，但在合适参数和场景下，"region 冷热 ≈ 内部 pages 冷热"的近似足以支撑内存管理决策（proactive reclaim、THP 迁移、冷内存检测等）。

---

## 参考来源

- [DAMON Design Document](https://docs.kernel.org/mm/damon/design.html) — Region Based Sampling & Adaptive Regions Adjustment
- [DAMON Usage Guide](https://docs.kernel.org/admin-guide/mm/damon/usage.html) — Monitoring Attributes & Region Tuning
- [SeongJae Park: DAMON tuning and results interpretation for hot pages (LKML, 2024-11)](https://lkml.iu.edu/hypermail/linux/kernel/2411.1/01384.html)
- [commit b9a6ac4e4ede — mm/damon: adaptively adjust regions](https://git.kernel.dk/cgit/linux/commit/mm?h=block-5.15-2021-09-17&id=b9a6ac4e4ede4172d165c133398b93e3233b0ba7)
- [commit 310d6c15e910 — mm/damon/core: merge regions aggressively when max_nr_regions is unmet](https://git.kernel.dk/cgit/linux/commit/mm/damon?id=310d6c15e9104c99d5d9d0ff8e5383a79da7d5e6)
- [RE: [PATCH v2 0/3] mm/damon: Profiling enhancements (accuracy discussion, 2024-03)](https://lkml.org/lkml/2024/3/22/1130)
