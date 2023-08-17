# 基于心跳的防沉迷设计

<figure><img src="../../.gitbook/assets/image (4) (2).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

1. SDK三分钟一次心跳，带着用户信息请求服务端http接口
2. 检查心跳用户实名制情况，未完成实名制返回指定code提示用户做实名制
3. 检查当前用户的在线信息，hash结构，如hash中不存在，表示是初次登录，发送一条Login记录给kafka，记录一条Signin记录。
4. 查询当前用户的Node，这里的Node代表Redis的账户集合的一条记录，对应一个Set结构，key是当前时间+心跳超时时间的年月日时分 + 第几个10秒，如2023080913021，value为用户信息。
5. 如果是初次登录，将当前用户加到对应的Node，如果是刷新心跳，从在线信息中找到LastHeartBeat时间，从而找到对应的Node，将用户信息从旧的Node中删除，并加入到新的Node。
6. 总结一下Node，Node就是一个账户集合，存储着10s的心跳用户信息。
7. 超时数组结构，分为成年人和未成年人两个redis zset结构，存储着用户的超时信息。key是玩家的游戏id，value是Node的key，score是超时时间。每10s内心跳都会刷新超时数组的信息。
8. 心跳处理完成，增加玩家的在线时长。在线信息中存储着一个身份证绑定的id的在线时长信息，取出数据 + (当前时间-上次心跳的时间)，刷新到hash结构中。
9. 未成年宵禁job扫描未成年zset，rangByScore找出所有未成年人信息，全部执行下线
10. 心跳超时job扫描未成年和成年job，下线所有心跳超时用户
11. 所有下线操作，发送一条SignOut消息到kafka
12. Kafka消费者消费SignIn和SignOut信息，加到LogSlot，用Lock的Condition，加满了以后批量上报的nppa。调用nppa接口进行分布式限流。



