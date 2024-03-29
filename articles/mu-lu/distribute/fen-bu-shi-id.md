---
description: id
---

# 分布式id

## Leaf

*   基于mysql的方案，多个Leaf节点共用一张数据库表，表里有自增id和业务id和步长，不同业务之间相互隔离，Leaf节点一次性获取一批号段

    问题:

    * 自增id容易暴露业务属性
    * 数据库宕机会造成服务不可用
* 双Buffer优化，用两个Buffer来存储号段，每次只用一个号段，等这个号段的剩余id只剩10%时，会主动去数据库加载下一批的号段到另外一个Buffer，待号段使用完毕后切换到另外一个Buffer读取号段
* 雪花算法:基于时间戳和workerId实现，需要依赖zookeeper，Leaf启动时需要去zk查询有没有注册过顺序节点，如果已经注册过了计算得到workerId，比较当前时间和节点的上传时间，如果小于节点上传时间表示发生了时钟回拨。然后得到该workerId下所有临时节点，计算临时节点下上传的时钟值，做平均值校验。

