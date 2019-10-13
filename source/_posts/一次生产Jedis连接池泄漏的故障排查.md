---
title: 一次生产Jedis连接池泄漏的故障排查
date: 2019-10-13 16:23:29
tags: Redis
---
## 概述
最近公司系统新上线，运行几天后生产的某个服务突然间出现了一堆Redis的连接池泄漏的错误日志，并且该服务再也无法使用调用redis，日志错误如下
![](/images/redis/jedis-error-log.png)

## 环境说明
项目采用的是Springboot，访问redis使用的是Spring Data Redis + Jedis

## 故障排查
从上面的错误日志可以看出这个错误是从连接池从拿不到连接，然后等待获取连接超时导致的。一开始，我们怀疑是不是访问量一多，连接池不够用导致的，但是在日志中我们发现出来一次报错后后面每次访问Redis都会报错，如果是连接池不够用，那应该是出现部分报错而不是全部，所以我们猜测是不是拿了连接后没有归还，导致连接池泄漏了。我们找到第一次出现错误的接口，接口访问 Redis 的代码如下
![](/images/redis/jedis-error-code.png)
这是一个获取 Redis 中数据数量的接口，该接口直接从 RedisTemplate 中获取连接池，然后再获取连接，使用后没有调用 close 方法，所以每次拿出来后都不会把连接归还，这个接口用的比较少，但是还是慢慢的把连接池给耗光了，所以在最后一次调用后就一直出现报错。

## 解决方式
- 1. 使用结束后调用连接池的 close 方法归还连接
- 2. 使用 RedisTemplate 访问，如下
```java
redisTemplate.execute(conn -> {
        return conn.dbSize();
}, true);
```
下面 execute 的实现
```java
@Nullable
public <T> T execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline) {
    try {
      ... 省略
    } finally {
        // 释放连接
        RedisConnectionUtils.releaseConnection(conn, factory, this.enableTransactionSupport);
    }
    return var11;
}
```
看下 releaseConnection 的实现
```java
public static void releaseConnection(@Nullable RedisConnection conn, RedisConnectionFactory factory, boolean transactionSupport) {
     if (conn != null) {
         RedisConnectionUtils.RedisConnectionHolder connHolder = (RedisConnectionUtils.RedisConnectionHolder)TransactionSynchronizationManager.getResource(factory);
         if (connHolder != null && connHolder.isTransactionSyncronisationActive()) {
             if (log.isDebugEnabled()) {
                 log.debug("Redis Connection will be closed when transaction finished.");
             }

         } else {
             // 使用事务的情况下
             if (isConnectionTransactional(conn, factory)) {
                 if (transactionSupport && TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                     if (log.isDebugEnabled()) {
                         log.debug("Unbinding Redis Connection.");
                     }

                     unbindConnection(factory);
                 } else if (log.isDebugEnabled()) {
                     log.debug("Leaving bound Redis Connection attached.");
                 }
             } else { // 没使用事务的情况
                 if (log.isDebugEnabled()) {
                     log.debug("Closing Redis Connection.");
                 }
                 // 关闭连接
                 conn.close();
             }

         }
     }
 }
```
Spring Data Redis 中默认是不开启事务的，如果开启事务的话那 Spring Data Redis 是不会自动释放连接，具体可以看这篇[Sping Data Redis 使用事务时，不关闭连接的问题](https://blog.csdn.net/LiuHanFanShuang/article/details/52136438)
