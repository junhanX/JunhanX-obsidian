# Linux 内核 DAMON 核心结构体

> 所有核心结构体定义在 `include/linux/damon.h`，源码目录在 `mm/damon/`。

---

## 1. 基础类型

### struct damon_addr_range — 地址范围

```c
struct damon_addr_range {
    unsigned long start;   // 起始地址（含）
    unsigned long end;     // 结束地址（不含）
};
```

### struct damon_attrs — 监控属性

```c
struct damon_attrs {
    unsigned long sample_interval;     // PTE 采样间隔（微秒）
    unsigned long aggr_interval;       // 聚合检查间隔
    unsigned long ops_update_interval; // 操作更新间隔
    unsigned long min_nr_regions;      // Region 最小数量
    unsigned long max_nr_regions;      // Region 最大数量
};
```

---

## 2. 监控层

### struct damon_region — 监控区域

```c
struct damon_region {
    struct damon_addr_range ar;      // 地址范围 [start, end)
    unsigned long sampling_addr;     // 下次采样地址
    unsigned int nr_accesses;        // 上一周期检测到的访问次数
    unsigned int nr_accesses_bp;     // 固定点精度版本
    unsigned int age;                // 连续未访问的聚合周期数
    struct list_head list;           // 链接到 damon_target 的 regions_list
};
```

### struct damon_target — 监控目标

```c
struct damon_target {
    struct pid *pid;              // 被监控进程的 PID
    unsigned int nr_regions;      // 该目标的 Region 数量
    struct list_head regions_list; // Region 链表头
    struct list_head list;        // 链接到 damon_ctx 的 adaptive_targets
};
```

### struct damon_ctx — 监控上下文（顶层结构）

```c
struct damon_ctx {
    struct damon_attrs attrs;          // 监控属性
    struct task_struct *kdamond;       // 内核采样线程
    struct mutex kdamond_lock;         // kdamond 保护锁
    struct damon_operations ops;       // 原语操作（vaddr / paddr / fvaddr）
    struct damon_callback callback;    // 回调钩子
    struct list_head adaptive_targets; // damon_target 链表
    struct list_head schemes;          // DAMOS scheme 链表
};
```

### struct damon_operations — 原语操作接口

```c
struct damon_operations {
    enum damon_ops_id id;
    // 生命周期
    void (*init)(struct damon_ctx *ctx);
    void (*update)(struct damon_ctx *ctx);
    void (*cleanup)(struct damon_ctx *ctx);
    // 核心监控循环
    void (*prepare_access_checks)(struct damon_ctx *ctx);    // ② Sample
    unsigned int (*check_accesses)(struct damon_ctx *ctx);   // ④ Check
    void (*reset_aggregated)(struct damon_ctx *ctx);         // 重置聚合
    // DAMOS 执行
    unsigned long (*apply_scheme)(struct damon_ctx *, struct damon_target *, struct damon_region *, struct damos *);
    // 其他
    int (*get_scheme_score)(struct damon_ctx *, struct damon_target *, struct damon_region *, struct damos *);
    bool (*target_valid)(struct damon_target *);
};
```

三种原语实现分别在：
- `mm/damon/vaddr.c` — 虚拟地址监控（用户态进程）
- `mm/damon/paddr.c` — 物理地址监控（系统级）
- `mm/damon/fvaddr.c` — 固定虚拟地址

---

## 3. DAMOS 策略层

### struct damos_access_pattern — 目标访问模式匹配

```c
struct damos_access_pattern {
    unsigned long min_sz_region;  // Region 大小下限
    unsigned long max_sz_region;  // Region 大小上限
    unsigned int min_nr_accesses; // 访问频率下限
    unsigned int max_nr_accesses; // 访问频率上限
    unsigned int min_age_region;  // Region age 下限
    unsigned int max_age_region;  // Region age 上限
};
```

### struct damos — 策略方案（Scheme）

```c
struct damos {
    struct damos_access_pattern pattern;  // 匹配条件
    enum damos_action action;             // 执行动作
    unsigned long apply_interval_us;      // 应用间隔
    struct damos_quota quota;             // 配额控制
    struct damos_watermarks wmarks;       // 水位线开关
    struct list_head filters;             // 过滤器链表
    struct damos_stat stat;               // 统计信息
    struct list_head list;                // 链接到 damon_ctx.schemes
};
```

### struct damos_quota — 配额（控制激进程度）

```c
struct damos_quota {
    unsigned long reset_interval;  // 配额重置周期
    unsigned long ms;              // 时间配额（毫秒）
    unsigned long sz;              // 大小配额（字节）
    unsigned long esz;             // 有效大小
    unsigned int weight_sz;        // 大小评分权重
    unsigned int weight_nr_accesses; // 访问频率评分权重
    unsigned int weight_age;       // Age 评分权重
    struct list_head goals;        // 自动调优目标
};
```

### struct damos_watermarks — 水位线

```c
struct damos_watermarks {
    enum damos_wmark_metric metric; // 度量指标（如 free mem rate）
    unsigned long interval;         // 检查间隔
    unsigned long high;             // 高于 high → 不执行
    unsigned long mid;              // mid 附近 → 正常执行
    unsigned long low;              // 低于 low → 不执行
};
```

### struct damos_filter — 过滤器

```c
struct damos_filter {
    enum damos_filter_type type;    // 过滤类型
    bool matching;                  // true=匹配则执行, false=匹配则跳过
    union {
        unsigned short memcg_id;    // memcg 过滤
        struct damon_addr_range addr_range; // 地址范围过滤
        int target_idx;             // 目标索引
    };
    struct list_head list;
};
```

### struct damos_stat — 统计

```c
struct damos_stat {
    unsigned long nr_tried;      // 尝试应用次数
    unsigned long sz_tried;      // 尝试处理的数据量
    unsigned long nr_applied;    // 实际成功应用次数
    unsigned long sz_applied;    // 实际处理的数据量
    unsigned long qt_exceeds;    // 超出配额的次数
};
```

---

## 4. 结构关系图

```
damon_ctx (每个 kdamond 线程一个)
 ├── attrs (sample/aggr_interval, min/max_nr_regions)
 ├── ops (vaddr/paddr/fvaddr 原语操作集)
 ├── adaptive_targets ──→ [damon_target]
 │                        ├── pid
 │                        ├── nr_regions
 │                        └── regions_list ──→ [damon_region]
 │                                               ├── ar [start, end)
 │                                               ├── sampling_addr
 │                                               ├── nr_accesses, age
 │                                               └── list
 │
 └── schemes ──→ [damos]
                 ├── pattern [min/max sz, nr_accesses, age]
                 ├── action (PAGEOUT/MIGRATE/HUGEPAGE/...)
                 ├── quota [ms, sz, weights]
                 ├── wmarks [high/mid/low]
                 ├── filters ──→ [damos_filter]
                 └── stat [nr/sz tried/applied]
```
