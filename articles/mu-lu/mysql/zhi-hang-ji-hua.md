# 执行计划

system>const>eq\_ref>ref>range>index>ALL

* system: 表里只有一条数据
* const: 主键或者唯一索引单表通过常量等值查询
* eq\_ref: 使用了索引的所有组成部分，并且索引是主键或者唯一索引
* ref: 主键或者唯一索引满足最左匹配原则，或者二级索引匹配少量的数据
* range: 单表通过二级索引范围查询
* index: 通过索引全表扫描
* All: 全表扫描
