问题1：统计数据一直为0
[root@hanzj-jumper kdamonds]# ls 0/contexts/0/schemes/0/stats/
nr_applied  nr_tried  qt_exceeds  sz_applied  sz_tried

[root@hanzj-jumper kdamonds]# cat  0/contexts/0/schemes/0/stats/*
0
0
0
0
0

低版本内核 混用 /sys/kernel/mm/damon/admin/kdamonds/0/state 
echo update_schemes_stats > /sys/kernel/mm/damon/admin/kdamonds/0/state 就可以看到统计数据了 

问题2：  /sys/kernel/mm/damon/admin/kdamonds/0/contexts/0/targets/0/regions/nr_regions 一直是0 

nr_regions=0 本身不是 bug，DAMON
  会自动扫描地址空间。如果要指定监控特定地址范围，才需要手动设置
  nr_regions 和对应的 start/end。


