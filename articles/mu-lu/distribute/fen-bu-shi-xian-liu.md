# 分布式限流

1.  滑动窗口限流

    可以用redis的Zset来实现滑动窗口限流，思路如下:

    假设限制m秒内只能处理n个请求，则先清除m秒之前的数据

```
long now = System.currentTimeMils();
redisTemplate.opsForZSet.removeRangeByScore(key, 0, now / 1000 - m);
```

&#x20;     然后判断剩下有多少条数据

```
long count = redisTemplate.opsForZSet.count(key, 0, now);
```

如果count值大于等于n，则返回false



2. 漏斗桶
   1. 用一个队列做漏斗，无论加入队列的速度有多快，消费队列的速度是恒定的，一旦队列加满就不再接受请求
3. 令牌桶
   1. 有一个恒定速度往桶里面放令牌，请求要到桶里拿到一个令牌才能通过请求。
