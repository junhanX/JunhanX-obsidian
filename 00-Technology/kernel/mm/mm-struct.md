# mm struct 学习


struct lruvec 代表内存的可回收的部分 



memcg 属于全局结构 服务于一个cgroup。管理的内存就可能会跨越多个node. memcg 跨node 那必然每个node下面都得有lruvec 

memcgA
 ├── node0--> lruvec
 ├── node1--> lruvec 
 └── node2 --> lruvec