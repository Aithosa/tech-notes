# 基于 redis 的分布式令牌桶算法

## 简介

本算法主要参考了 Google 的 Guava 令牌桶的实现，不过由于 Guava 是单机算法，无法和其他服务器同步令牌桶状态。

<!--
Demo 代码见[RateLimiter](https://codehub-g.huawei.com/w00585603/RateLimiter/files?ref=master)
移植到 gw-air 的代码见[f_rate_limiter 分支](https://codehub-g.huawei.com/smartcom/itravel/traffic/gw-air/files?ref=f_rate_limiter) 。
-->

> f_rate_limiter 分支精简并优化了代码，替换 jedis 为 redisTemplate，替换 Redission 为 Itrdk 实现的分布式 Redis 锁。

本算法把基本参数分为**配置参数**以及**状态参数**，配置参数保留在令牌桶对象中，状态参数保存在 Redis 中，每次操作需要到 Redis 中读取状态参数，使得使用同一令牌桶的客户端能共享令牌桶的状态。

状态参数完全由配置参数生成，即使状态参数丢失了，也可以根据配置参数重新生成。

令牌桶的两个主要的方法分别是`acquire()`和`tryAcquire()`。

- `acquire()`方法尝试获取一个(或指定数量)的令牌，若桶中没有足够的令牌，则会等待足够数量的令牌生成。
- `tryAcquire()`方法尝试获取一个(或指定数量)的令牌，同时可提供一个超时时间，若所需令牌可以在超时时间内获取，则会等待相应时间并获取令牌，否则返回失败。

RateLimiter 主要结构如下图：
![RateLimiter主要结构](https://cyber-reed.tech/usr/uploads/2022/09/2378969169.png)

在使用时，需要确保涉及的服务器连接的是同一个 Redis，操作同一个令牌桶状态参数。如下图：
![RateLimiter部署](https://cyber-reed.tech/usr/uploads/2022/09/2368370686.png)

## 基本数据结构

令牌桶通过配置生成，配置类的结构同令牌桶的配置参数，具体如下：

```java
/**
 * 唯一标识，令牌桶的名字，用于查找
 */
private final String name;

/**
 * 每秒存入的令牌数
 */
private final long permitsPerSecond;

/**
 * 最大存储令牌数
 */
private final long maxPermits;

/**
 * 缓存比例(相对于桶大小或者每秒钟生成的令牌数量)
 */
private final float cache;

/**
 * 分布式互斥锁
 */
private final RLock lock;

/**
 * 用于对Redis进行读取和查找操作
 */
private final RedisService redisService;
```

在配置参数中:

- 令牌桶名称`name`用于在本地查找令牌桶对象，并生成存储在 Redis 的查找状态参数的 key。目前以 IBE+ 的 Url 后缀命名。
- 每秒存入的令牌数`permitsPerSecond`用于计算单个令牌生成的时间间隔。
- 缓存比例`cache`是之所以选择这个实现的关键。令牌桶每秒钟生成的令牌数是固定的，但是允许缓冲区的存在，如每秒钟产生 5 个令牌，当 5 个令牌全都发出去了之后，并不会拒绝这一秒内的其他令牌申请，而是允许若干申请在超时时间内等待。缓存比例则决定了可以等待的申请有多少。_其他实现虽然也有缓存实现，但缓存区大小只有 1。_

> 对于本次需求，每秒存入的令牌数与最大存储令牌数取值相同。如 IBE+ 某个接口每秒允许调用 5 次，则每秒生成 5 个令牌，且桶最大只能存储 5 个令牌。_后面调参时可以考虑是否改变。_

创建令牌桶之后，会初始化状态参数，状态参数类名为 `PermitBucket`，其主要成员如下：

```java
/**
 * 唯一标识，令牌桶的名字，用于查找
 */
private String name;

/**
 * 最大存储令牌数
 */
private long maxPermits;

/**
 * 当前存储令牌数
 */
private long storedPermits;

/**
 * 每两次添加令牌之间的时间间隔，单位为纳秒
 */
private long intervalMicros;

/**
 * 下一个获取令牌请求被批准的时间
 */
private long nextFreeTicketMicros;
```

在状态参数中:

- 当前存储令牌数`storedPermits`用于存储桶中当前的令牌数，令牌允许透支，但所保存的当前令牌数不会为负。
- 每两次添加令牌之间的时间间隔`intervalMicros`通过配置信息计算而来，用于读取同状态后计算这段时间生成的令牌数。
- 下一个获取令牌请求被批准的时间`nextFreeTicketMicros`基于**时间戳**，存储的是达到平衡状态的时间，即达到下一个不欠任何令牌状态的时间。若所存储的时间戳>当前时间的时间戳，则表明令牌桶存在负债。

> 令牌桶定义了一个**平衡状态**的概念，因为本算法允许令牌透支，而令牌桶中的令牌数量最低保存 0，透支主要体现在`nextFreeTicketMicros`中，每欠一个令牌，该字段值就会加上生成一个令牌的时间。定义`nextFreeTicketMicros`里存储的时间即位令牌桶恢复平衡状态的时间，即该桶不欠任何令牌的时间，到达该时间即令牌桶不再欠任何令牌。

## 桶状态类 PermitBucket 方法

令牌桶状态类有一个方法`reSync`，负责在从 Redis 读取到桶的状态信息之后更新桶的状态。具体就是根据令牌生成间隔`intervalMicros`和下一次达到平衡状态的时间戳`nextFreeTicketMicros`向桶里放入令牌。具体实现如下：

```Java
public void reSync(long nowMicros) {
    // 当前时间大于下一个获取令牌请求被批准的时间，才会执行更新
    if (nowMicros > nextFreeTicketMicros) {
        // 这段时间生成的实际能放入桶中的令牌数
        long newPermits = (nowMicros - nextFreeTicketMicros) / intervalMicros;
        storedPermits = min(maxPermits, storedPermits + newPermits);
        // 如果时间还不够生成新的令牌，不需要更新nextFreeTicketMicros
        if (newPermits > 0L) {
            // 这样可能会带来误差，但是也会定期校准时间
            nextFreeTicketMicros = nowMicros;
        }
    }
}
```

## 令牌桶核心方法

`RateLimiter`类的两个核心方法分别是`acquire()`和`tryAcquire()`。

### `acquire()`

`acquire()`的特点是缓冲队列无限长，调用了这个方法就一定会取到令牌，不管需要等待的时间有多长。若没有参数，则默认获取一个令牌。方法会返回为了获得令牌而等待的时间。其大体实现如下：

```Java
public double acquire(int permits) {
    // 获取所需的令牌需要等待的时间
    long microsToWait = reserve(permits);
    // 等待获取令牌期间休眠
    sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
}

private long reserve(int permits) {
    checkPermits(permits);
    while (true) {
        if (lock()) {
            try {
                return reserveAndGetWaitLength(permits, MILLISECONDS.toMicros(System.currentTimeMillis()));
            } finally {
                unlock();
            }
        }
    }
}
```

`reserve()`不只返回了获取令牌需要等待的时间，同时也把这段时间加到了 nextFreeTicketMicros 上。

`reserve()`的最终执行方法如下:

```Java
private long reserveEarliestAvailable(long requiredPermits, long nowMicros) {
    PermitBucket bucket = getBucket();
    bucket.reSync(nowMicros);

    // 结合这次请求，当前总共能提供出去的令牌数
    long storedPermitsToSpend = min(requiredPermits, bucket.getStoredPermits());
    // 这次请求还欠的令牌数
    long freshPermits = requiredPermits - storedPermitsToSpend;
    // 生成还欠的令牌数需要花的时间
    long waitMicros = freshPermits * bucket.getIntervalMicros();

    bucket.setNextFreeTicketMicros(saturatedAdd(bucket.getNextFreeTicketMicros(), waitMicros));
    long returnValue = bucket.getNextFreeTicketMicros();
    // 这里不为负，最多为0，后面会休眠负令牌清零的时间，等待令牌恢复
    bucket.setStoredPermits(bucket.getStoredPermits() - storedPermitsToSpend);
    setBucket(bucket);

    return returnValue;
}
```

解释：

在从 Redis 取到桶状态之后会同步桶的状态，即放入这段时间生成的令牌。随后会根据现有的令牌数计算是否会产生负债及负债后到达平衡状态的时间`NextFreeTicketMicros`，最后返回到达平衡状态的时间，用于计算本次需要休眠多久恢复平衡状态。

- 当桶中令牌足够时，获取令牌的请求会立刻通过，returnValue 为 0，不需要休眠等待。
- 当桶中令牌数量不足时，因为缓冲队列无限长，所以会计算如果这次请求通过，当前的令牌桶会产生多少令牌负债，然后把本次请求产生的负债令牌生成的时间加到`NextFreeTicketMicros`上。这次请求不会被立刻通过，而是要从当前时刻开始，休眠到此时的新算出的平衡时间。表示休眠结束时，这次请求以及之前请求所欠的令牌已经全部恢复了(若后面还有请求，并且不会超时，后面请求也会如此休眠)。

### `tryAcquire()`

`tryAcquire()`的缓冲队列长度是由超时时间决定的，其参数有`long permits, long timeout, TimeUnit unit`。其中`timeout`表示此次调用允许等待最多多久。与其他实现不同的是，超时时间内允许缓冲多个请求，即若第一个请求不会超时，进入等待后，下一个请求如果继续等待也不会超时，允许往缓冲队列继续追加请求。实现的原理就是利用了`NextFreeTicketMicros`，允许缓冲的请求就把需要花费的时间加到`NextFreeTicketMicros`上并等待。`tryAcquire()`的实现如下：

```Java
public boolean tryAcquire(long permits, long timeout, TimeUnit unit) {
    checkPermits(permits);
    long timeoutMicros = max(unit.toMicros(timeout), 0);
    long waitMicros = 0L;
    while (true) {
        if (lock()) {
            try {
                long nowMicros = unit.toMicros(System.currentTimeMillis());
                // 判断这次请求是否会导致超时
                if (!canAcquire(permits, nowMicros, timeoutMicros)) {
                    return false;
                } else {
                    // 不会超时的话就预约并获取等待的时间
                    waitMicros = reserveAndGetWaitLength(permits, nowMicros);
                }
            } finally {
                unlock();
            }
            sleepMicrosUninterruptibly(waitMicros);
            return true;
        }
    }
}
```

在收到一个`tryAcquire()`请求后`canAcquire()`会先同步一次令牌桶的状态，补充当前可生成的令牌。随后判断这次的请求会不会导致超时，即这次请求需要消耗掉的待生成令牌的时间加到`NextFreeTicketMicros`上之后会不会超时，相当于预计算，预计算的结果不会写到状态参数上去，若不会超时则在`reserveAndGetWaitLength()`正式预约令牌并更新`NextFreeTicketMicros`。预计算的流程和`reserveEarliestAvailable()`基本相同，只是没有更新令牌桶状态信息。

## 其他细节

### 各接口配置数据

配置信息存放于 gw-air 的 Mysql 表格`sgp_igateway.t_ratelimite_conf`：

```sql
create table t_ratelimite_conf
(
    configId       INT(11)              NOT NULL AUTO_INCREMENT,
    channelType    VARCHAR(24)          NULL,     # 供应商(目前只有IBE+)
    interfaceNo    VARCHAR(24)          NULL,     # 接口编号(IBE)
    interfaceUrl   VARCHAR(128)         NULL,     # 接口URL, 会用于Redis键值
    rateLimit      INT(11)              NULL,     # 并发数限制
    cache          FLOAT(3, 2)          NULL,     # 缓冲区(暂定比例)
    status         TINYINT(1) DEFAULT 0 NOT NULL, # 配置状态 0 禁用, 1 启用
    lastModifyTime datetime   DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`configId`)
);
```

其中`interfaceUrl`是各个接口令牌桶的 name；`rateLimit`为调用次数限制；`cache`为缓冲区比例，默认为 0.5，及允许超时等待最多半秒；status 为当前配置是否开启。

### 限流开关

本次 IBE+限流有一个 Apoll 配置的全局开关，考虑到令牌的获取都是通过`tryAcquire()`方法，因此在调用这个方法的时候会判断全局开关的状态，如果开关是关闭的则会返回`true`，不需要限流。

```Java
/**
 * 以毫秒为单位, 使用预设值的timeout, 考虑开关状态
 */
public boolean tryAcquire(String switchConf) {
    if (Constant.NO.getCode().equals(switchConf)) {
        return true;
    }
    return tryAcquire(1, timeout, TimeUnit.MILLISECONDS);
}
```

### 分析

以上两种情况都可以归类为令牌桶冷启动问题。

当桶内令牌数较多(尤其是满令牌时)，对于突发请求会出现实际一秒内发出的令牌数量多于期望的情况。

例如：创建一个限流器，每秒发送 5 个令牌，桶的大小也设置为 5，等待令牌桶装满 5 个令牌，假设每个请求申请一个令牌，在极短的时间内先来了 5 个请求，此时令牌桶中的 5 个令牌会全部发出去，假设处理折 5 个请求花了 50ms，则 1 秒中还剩下 950ms，这段时间最多还能生成 4 个令牌，因此这 1 秒中最多能发出去 9 个令牌。

在实际测试的情况下，连接远程的 redis，并且代码运行速度稍微慢一些的话，就会出现这种情况。比如在令牌桶满令牌的情况下来了 10 个请求，光是把桶里的 5 个令牌发出去就会花费超过 200ms，此时已经新生成了一个令牌，假设超时时间还允许透支 3 个令牌，则一秒内实际发出了 5+1+3=9 个令牌。

对于这种情况，有两种解决方法：

1. 减小令牌桶的容量，令牌生成的速度不变
2. 改进令牌桶算法，参考 Guava 的`SmoothWarmingUp`实现，增加从桶里拿走已经生成的令牌的代价。

### 一些 SmoothWarmingUp 里的关键函数

```java
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
    double oldMaxPermits = maxPermits;
    double coldIntervalMicros = stableIntervalMicros * coldFactor;                // @1
    thresholdPermits = 0.5 * warmupPeriodMicros / stableIntervalMicros;    // @2
    maxPermits = thresholdPermits + 2.0 * warmupPeriodMicros / (stableIntervalMicros + coldIntervalMicros);   // @3
    slope = (coldIntervalMicros - stableIntervalMicros) / (maxPermits - thresholdPermits);  // @4
    if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        storedPermits = 0.0;
    } else {
        storedPermits = (oldMaxPermits == 0.0)
                        ? maxPermits // initial state is cold
                        : storedPermits * maxPermits / oldMaxPermits;    // @5
    }
}


/**
 * 正常是0，不需要等，只计算当此还欠的令牌生成的时间
 * 本函数在其基础上加了一段时间
 * storedPermits 桶中目前存储的令牌
 * permitsToTake 这次请求能直接提供出去的令牌
 */
long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
    // 当前存储的令牌多于阈值的数量
    // 如果超过 thresholdPermits ，申请许可将来源于超过的部分，只有其不足后，才会从 thresholdPermits 中申请
    double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
    long micros = 0;
    // measuring the integral on the right part of the function (the climbing line)
    // 当前存储的令牌多于阈值
    if (availablePermitsAboveThreshold > 0.0) {
        // 获取本次从预热区间申请的许可数量，优先从阈值以上部分取
        double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);

        double length = permitsToTime(availablePermitsAboveThreshold)
                        + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);

        micros = (long) (permitsAboveThresholdToTake * length / 2.0);

        // 这次请求还需要从阈值以下取的令牌
        permitsToTake -= permitsAboveThresholdToTake;
    }

    // 阈值以上取的令牌等待的时间，加上阈值以下取的令牌等待的时间
    micros += (long) (stableIntervalMicros * permitsToTake);

    return micros;
}

private double permitsToTime(double permits) {
    return stableIntervalMicros + permits * slope;
}
```

## 参考

### 参考 blog

https://blog.csdn.net/Victorgcx/article/details/104248819
https://github.com/Augustvic/LIMIT

### 参考代码

redisson https://github.com/redisson/redisson
