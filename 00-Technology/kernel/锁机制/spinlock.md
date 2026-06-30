自旋锁 ---》 spinlock 

特点：
- 获取不到锁就一直循环
- 不允许调度
- 不允许睡眠
- 持锁的时间必须短


spinlock 在6.18内核中的实现路径

spinlock --> raw_spinlock--> arch_spinlock --> qspinlock实现
```
// atomic_t 32位的内存
// 32 bit 被后续的 struct定义为以下结构
//
//+--------+--------+----------------+
//| locked | pending|      tail      |
//+--------+--------+----------------+
//  8 bit    8 bit       16 bit
typedef struct qspinlock {
    union {
        atomic_t val;

        /*
         * By using the whole 2nd least significant byte for the
         * pending bit, we can allow better optimization of the lock
         * acquisition for the pending bit holder.
         */
#ifdef __LITTLE_ENDIAN
        struct {
            u8  locked;
            u8  pending;
        };
        struct {
            u16 locked_pending;
            u16 tail;
        };
#else
        struct {
            u16 tail;
            u16 locked_pending;
        };
        struct {
            u8  reserved[2];
            u8  pending;
            u8  locked;
        };
#endif
    };
} arch_spinlock_t;

```

|字段|大小|作用|
|---|---|---|
|`locked`|8 bit|表示锁是否已被持有|
|`pending`|8 bit|表示"有人正在尝试获取锁"，是 Fast Path 和 Slow Path 之间的过渡状态|
|`tail`|16 bit|保存 MCS 等待队列的尾节点信息（编码后的 CPU/node 信息）|
