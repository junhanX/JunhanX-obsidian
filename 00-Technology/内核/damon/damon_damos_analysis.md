# Linux DAMON & DAMOS 技术分析与实践指南

> `DAMON（监控引擎）` → `DAMOS（策略执行层）` → `damo（用户态工具）`

## 1. 概述

**DAMON**（Data Access Monitor）是 Linux 内核中用于高效监控内存访问模式的框架。它不管理内存，只回答一个问题：**"哪些页面热、哪些页面冷？"** 然后由 DAMOS 根据答案自动执行操作。

核心只做三件事：

1. **监控**：低开销采样 PTE Accessed bit，CPU 开销 < 3%
2. **识别**：自适应 Region 粒度，TB 级内存可用
3. **触发策略**：用户声明"冷/热时该做什么"，内核自动执行

```
  kdamond ──采样内存──→ DAMON(识别冷热) ──触发Scheme──→ DAMOS(执行层)
   │                      │                              │
   └──────────────────────┴──────────────────────────────┘
                                          ↓
     PAGEOUT / MIGRATE / THP / LRU_SORT / STAT / WILLNEED / …
           ↑                              ↑
    damo/sysfs配置方案               内核自动执行
```

---

## 2. 核心架构

DAMON 通过 `kdamond` 内核线程执行周期性采样：

```
  ① Regions ──→ ② Sample(PTE) ──→ ③ Wait(N cycles)
     ↑                                      │
     └── ⑤ Adapt(hot→shrink,cold→grow) ←── ④ Check
```

**四大设计原则**：精确（Region 级粒度）· 轻量（硬件 bit + 采样）· 可扩展（自适应粒度，TB 级内存可用）· 可调（运行时调参数）

---

## 3. DAMOS 机制

DAMOS 是 DAMON 框架的**策略执行层**。根据监控结果，自动执行预定义的内核操作。

> DAMOS 随 `CONFIG_DAMON` 一起启用，**不存在独立的 `CONFIG_DAMOS`**。

### 3.1 Action 分类一览

| 分类      | Action               | 底层实现                       | 典型场景                  |
| ------- | -------------------- | -------------------------- | --------------------- |
| **回收**  | `DAMOS_PAGEOUT`      | `madvise(MADV_PAGEOUT)`    | 立即换出冷页，释放 DRAM        |
|         | `DAMOS_COLD`         | `madvise(MADV_COLD)`       | 标记冷页，移入非活跃 LRU        |
| **分层**  | `DAMOS_MIGRATE_HOT`  | 跨 NUMA 迁移                  | 热页提升至 DRAM            |
|         | `DAMOS_MIGRATE_COLD` | 跨 NUMA 迁移                  | 冷页降级至 CXL             |
| **LRU** | `DAMOS_LRU_PRIO`     | 移向活跃 LRU 头部                | 保护热页不被回收              |
|         | `DAMOS_LRU_DEPRIO`   | 移向非活跃 LRU 尾部               | 预置冷页等待回收              |
| **大页**  | `DAMOS_HUGEPAGE`     | khugepaged 合并              | 热区域合并 THP，减少 TLB Miss |
|         | `DAMOS_NOHUGEPAGE`   | `madvise(MADV_NOHUGEPAGE)` | 防止稀疏区域误合并             |
| **其他**  | `DAMOS_WILLNEED`     | `madvise(MADV_WILLNEED)`   | 预读即将访问的数据             |
|         | `DAMOS_STAT`         | 计数器                        | 仅统计，无内存操作             |

### 3.2 Scheme 结构

一个 scheme 定义"**什么条件下执行什么动作**"：

```
  Condition          Action           Quota
─────────────────────────────────────────────────
  Age(N cycles)    →  Action        →  ms(time)
  Watermark(mem)      (pageout/migrate)    sz(size)
  Filter(anon/memcg/NUMA/hugepage)
```

**触发逻辑**：`Watermark 通过` AND `Filter 匹配` AND `Age 满足` → 执行 Action（受 Quota 限制）

### 3.3 源码执行路径

```
kdamond_fn()
  → damon_do_access_check()    // 采样：清除 PTE Accessed bit
  → damon_aggregate()          // 聚合：检查 Accessed bit
    → damon_apply_scheme()     // 遍历 DAMOS scheme
      → damon_check_watermark()// 水位线检查
      → damon_filter()         // 过滤器匹配
      → damos_apply_action()   // PAGEOUT / MIGRATE / …
```

---

## 4. 内核配置

所有 DAMON 配置选项位于 `mm/damon/Kconfig`。

| 分类 | 配置项 | 说明 |
|------|--------|------|
| **核心** | `CONFIG_DAMON` | 核心框架（必须） |
| **原语** | `CONFIG_DAMON_PADDR` / `CONFIG_DAMON_VADDR` | 物理地址 / 虚拟地址监控 |
| **接口** | `CONFIG_DAMON_SYSFS` | 现代接口（`/sys/kernel/mm/damon/`） |
| | `CONFIG_DAMON_DBGFS` | 旧版 debugfs（已废弃） |
| **预置策略** | `CONFIG_DAMON_RECLAIM` / `CONFIG_DAMON_LRU_SORT` | 内核预置的回收/排序策略 |
| **统计** | `CONFIG_DAMON_STAT` | 轻量级访问统计 |

```bash
# 最低可用配置
CONFIG_DAMON=y  CONFIG_DAMON_VADDR=y  CONFIG_DAMON_PADDR=y  CONFIG_DAMON_SYSFS=y

# 检查当前内核
grep -i damon /boot/config-$(uname -r)
ls /sys/kernel/mm/damon/
dmesg | grep -i damon
```

## 5. 应用场景

### 5.1 三级阶梯模型

在有 CXL 且磁盘充足的系统中，三种机制组合成完整的三级互补链路：

```
快 ──────────────────────────────────────────────── 慢
│                                                  │
DRAM ──MIGRATE_COLD──→  CXL ──PAGEOUT──→  NVMe Swap
 ~80ns          ~150ns           ~50µs
 (热)          (温/冷)           (极冷)
```

**数据流动闭环**：
1. **DRAM → CXL**：冷数据沉淀到 CXL，释放快速内存。
2. **CXL → Swap**：CXL 水位过高时，最冷数据换出到 NVMe。
3. **Swap/CXL → DRAM**：重新访问时，按需快速读回 DRAM。

**典型配置**：

```bash
# 热页提升 → 冷页降级 → 极冷页换出
damo schemes --acts migrate_hot  --target 0 --access 80-100 --age 0-2  --quota ms 10000 sz 268435456
damo schemes --acts migrate_cold --target 1 --access 0-0    --age 10- --quota ms 10000 sz 268435456
damo schemes --acts pageout      --access 0-0    --age 30- --quota ms 2000   sz 104857600
```

无 CXL 时去掉中间层：

```bash
damo schemes --acts pageout --access 0-0 --age 10- --quota ms 2000 sz 104857600
# 配合：sysctl -w vm.swappiness=200
```

### 5.2 CXL 分层 vs NVMe 回收

核心差异不在于"谁更快"，而在于**数据移出后是否仍在物理地址空间中**：

| 维度 | CXL 分层 | NVMe Swap 回收 |
|------|----------|----------------|
| 数据去向 | 仍在物理内存（跨 NUMA 节点） | 离开物理内存，写入磁盘 swap |
| PTE 状态 | 页表有效，指向新 NUMA 节点物理地址 | 页表清空，替换为 swap entry |
| 再次访问 | 透明 NUMA 远程读（~150ns），无阻塞 | Major Page Fault（~50µs+），线程挂起 |
| 内核操作 | `migrate_pages()` 内存拷贝 | `swapout`/`swapin` 块设备 IO |
| 适用场景 | 短期冷热波动（透明访问） | 长期容量压力（容量无限） |

**冷热循环**：冷页随时可以变回热页。DAMON 的 `age` 计数器基于**连续**未访问——中间只要有一次访问，`age` 立即清零重置。数据不会被贴上永久标签。

| 场景 | 重新访问路径 |
|------|-------------|
| **分层** | 透明 NUMA 远程读（~150ns）→ `age` 重置 → DAMON 检测到频率上升 → `MIGRATE_HOT` 迁回 DRAM |
| **回收** | Major Page Fault → 从 NVMe 读回 DRAM（~50µs+）→ `age` 重置 → 重新判定为热页 |

## 6. 最佳实践

1. **配额是生命线**：限制单次换出量（如 `sz 50M`），观察 `iostat %util` 和 `await` 微调。
2. **监控缺页频率**：Major Fault 频繁说明换出太快，调大 `--age` 让数据驻留更久。
3. **利用 Memcg**：容器化环境中为不同容器配置独立策略，避免互相影响。
4. **工具优先**：使用 [damo](https://github.com/damonitor/damo) 管理 DAMON/DAMOS，避免手动操作 sysfs。damo工具二分法确认版本 3.1.5
5. **大页迁移注意**：迁移前需 split 为小页，迁移后再 collapse。新内核提供 `hugepage_size` filter 优化
