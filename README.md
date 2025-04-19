# 背景
prometheus提供了promql这个强大而易于学习的工具，使得基于时序数据的查询变得简单，但是很多时候也带来了很多困扰。比如下面这个例子
```arkts
iops_ecs / iops_ecs_threshold > 90
```
