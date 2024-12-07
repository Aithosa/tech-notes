# Redis 中使用键空间监听 key 过期消息

作者: 敖天羽 时间: August 26, 2024 分类: Go

距离上一次更新已经超过一个月了，是月更博主对不起大家了！

主要是因为之前有一阵子业务比较忙，因此一直在加班，没有空看其他的东西（又不愿意牺牲打游戏和看剧的时间），最近一有时间就在写 Demo，这几天刚写完，才能更新这篇文章。

## 背景故事

这个需求也是在我们业务落地过程中衍生出来的，因此先来说说之前一阵子忙的东西吧。

在公司内做的服务因为有各种基建的加持，所以想要实现一些功能很容易，比如说标题写的东西，或者是 binlog 订阅消费；但是在 to B 私有化部署的场景下，客户机千奇百怪，就要求我们用尽可能少的依赖和简单的部署架构进行实现，肯定也不会有公司里这么多花里胡哨的依赖。

为此简化了不少架构和功能，牺牲了不少体验之后才给接入我们基建的用户怼上一个版本。

而其中一个诉求就是我们的功能需要（Nice to have）订阅过期键并广播给订阅用户。

## 需求分析

要满足这个需求，就需要一个支持下面两种能力的程序：

1. 能够订阅过期键事件
2. 能够确保消费的可靠性
   理论上来说，只要能有过期时间监听，那么就能跟 MySQL binlog 订阅一样进行数据消费，而 Redis 确实有个叫 keyspace notifications（也就是标题中的键空间通知），可以为 Redis 变更推送事件。

但众所周知，我们写的不是 Demo，而是生产需要落地的一个功能。为此，我们需要一定程度上保障通知的可达性。

在键空间通知里明确表明：

> Note: Redis Pub/Sub is _fire and forget_; that is, if your Pub/Sub client disconnects, and reconnects later, all the events delivered during the time the client was disconnected are lost.

换言之，如果我们直接使用，那么你服务宕机了，发版了，可能中间的数据就会永久性丢失，这一部分通知也就不到位了。

因此我们需要设计一个补偿措施，来保证「即使我的服务挂了，之后依旧能够补偿通知挂了期间的这一部分消息」。

## 想法：设计补偿

在过去的研发中，其实我们也设计过很多补偿策略了，比如事务回滚和重试。尤其是重试，诸如放进一个重试队列、过段时间再执行之类的思路是常规解法。

但是在 Redis 中却和 MySQL 的 binlog 有着明显的区别：他并没有一个持久化可读的内容——MySQL 中的 binlog 本质上是个文件，而文件就决定了，就算我失败了，那从失败行重新读就行了。

而 Redis 通知直接丢了，也不会帮我放进一个池子里，那么就需要自己实现一个持久化的池子，才能进行「补偿」操作。

当然，因此 expire 触发的时间点是不能抢救了，那么能抢救的就只有 key 入库的时间，假设 key 入库时：`SET key value EX [duration]`，那么我们默认在 `[duration]`时间后 key 会过期，且会发送通知。

那么要设计补偿，理论上我只需要一个定时任务，把一定时间内的未通知 key 捞出来过一遍就可以了。

## 开干：队列设计

本着多快好省不引入外部依赖的思路，对于队列，我们同样使用 Redis 来存储队列（本来是想用 MQ 的，写 Demo 又费劲，又不符合私有化部署降本的策略）。在 SET 的同时需要存储一条记录到队列，而其结构中应该包括：key 和到期时间两个。

从逻辑的角度来说，key 过期监听的补偿=该到期的 key - 已经成功发送通知的 key。

从这个角度来看，我们每次只要把 expireTime < now 的值捞出来就可以了。而 Redis 的 zset 正好可以匹配这一诉求，我们把 key 作为值，time 作为 score，这样通过 zrange 就可以很方便的捞出时间序并移除。

## 实现

有了监听、有了补偿，我们就可以开始进行实现了。

## Redis 配置

首先是最基本的 Redis 配置，要想让 Redis 支持消息推送，需要再 redis 的配置（redis.conf）中加入（或者找到被注释的行）notify-keyspace-events，文档中写明了可以配置的值：

```conf
K     Keyspace events, published with __keyspace@<db>__ prefix.
E     Keyevent events, published with __keyevent@<db>__ prefix.
g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
$     String commands
l     List commands
s     Set commands
h     Hash commands
z     Sorted set commands
t     Stream commands
d     Module key type events
x     Expired events (events generated every time a key expires)
e     Evicted events (events generated when a key is evicted for maxmemory)
m     Key miss events (events generated when a key that doesn't exist is accessed)
n     New key events (Note: not included in the 'A' class)
A     Alias for "g$lshztxed", so that the "AKE" string means all the events except "m" and "n".
```

对于只需要订阅过期消息的我们来说只要写：`notify-keyspace-events Ex` 就可以了。（然后记得重启）

当然，你也可以使用 redis-cli 来启用，类似于这样：`redis-cli -h <node-host> -p <node-port> config set notify-keyspace-events Ex`

## 监听实现

这里都以 Golang 为例，完整的代码可见：https://github.com/csvwolf/go-redis-watcher-demo/tree/master

监听的实现主要靠 pubsub，一个简单的 Demo 例子类似于：

```Go
func Watch(ctx context.Context) {
    // __keyevent@0__:* 监听的空间，也可以是 *__:*
    pubsub := w.redisClient.PSubscribe(ctx, "__keyevent@0__:*")
    defer pubsub.Close()
    ch := make(chan struct{}, 10)
    for {
        msg, err := pubsub.ReceiveMessage(ctx)
        if err != nil {
            fmt.Println("Error receiving message:", err)
            continue
        }
        ch <- struct{}{}
        go func(msg *redis.Message) {
            // 提取事件类型
            eventType := strings.TrimPrefix(msg.Channel, "__keyevent@0__:")
            // 根据事件类型做不同处理
            switch eventType {
            case "expired":
                fmt.Printf("Key expired: %s\n", msg.Payload)
                // 在这里处理过期事件
            case "del":
                fmt.Printf("Key deleted: %s\n", msg.Payload)
                // 在这里处理删除事件
            case "set":
                fmt.Printf("Key set: %s\n", msg.Payload)
                // 在这里处理设置事件
            default:
                fmt.Printf("Unhandled event %s for key %s\n", eventType, msg.Payload)
            }
            <-ch
        }(msg)
    }
}
    }
```

这里用 go routine 适当提升吞吐处理，这是个最简单的版本，只是用于验证你的 Redis 确实正常启用了键空间。

之后我们将持续优化并且将之前想到的各个部分补充上去。

## 补偿队列

上面只做了最基本的监听，根本没有结合「补偿」，根据我们之前的想法，构造一个 zset，因此我们封装了一个全新的 Set（最佳的实现来看，这里最好是事务的，也就是用 Lua 脚本，这里图省事就简单示意一下）：

```Go
func (r *RedisClient) Set(ctx context.Context, key string, value interface{}, expire time.Duration) error {
    var (
        err error
        now = time.Now().Add(expire)
    )
    if err = r.client.Set(ctx, key, value, expire).Err(); err != nil {
        return err
    }
    if expire == 0 {
        return nil
    }
    // 队列用于补偿
    if err = r.client.ZAdd(ctx, 'queue', redis.Z{Member: key, Score: float64(now.UnixMilli())}).Err(); err != nil {
        return err
    }
    return nil
}
```

同时，我们还需要设计获取和移除补偿队列的内容：

```Go
func (r *RedisClient) GetKeysByTime(ctx context.Context, endTime time.Time) ([]string, error) {
    return r.client.ZRangeByScore(ctx, r.queueKey, &redis.ZRangeBy{Min: "0", Max: strconv.FormatInt(endTime.UnixMilli(), 10)}).Result()
}
func (r *RedisClient) RemoveFromQueue(ctx context.Context, members ...interface{}) *redis.IntCmd {
    return r.client.ZRem(ctx, r.queueKey, members...)
}
```

这里本质上我们就不用在意执行失败了，因此假设获取失败了，那下一个 tick 再获取就好了，只是补偿时间延长了。而如果删除失败了，更不是问题，只要你的操作是幂等的就行。

最终我们得到了一个补偿任务的所有内容：

```Go
type EventType string
const (
    Expired EventType = "expired"
    Del     EventType = "del"
    Set     EventType = "set"
)
type Callback = func(action EventType, key string)
// MakeUpTask 补偿任务
func (w *Watcher) makeUpTask() {
    // 补偿在10秒前的所有值
    var (
        timer   = time.Now().Add(-10 * time.Second)
        members []interface{}
    )
    result, err := w.redisClient.GetKeysByTime(w.ctx, timer)
    if err != nil {
        fmt.Printf("Watcher makeup failed: err=%v", err)
        return
    }
    if len(result) == 0 {
        return
    }
    fmt.Println(time.Now().String(), "补偿 keys:", result)
    for _, r := range result {
        // callback：需要执行的处理函数
        w.callback(Expired, r)
        members = append(members, r)
    }
    err = w.redisClient.RemoveFromQueue(w.ctx, members...).Err()
    if err != nil {
        fmt.Printf("Watcher makeup failed: err=%v", err)
    }
    return
}
```

在这里我们定义了一个 10s 的时间，虽然这是一个拍脑袋定的值，但是请注意：

Redis 发送事件的事件是 Redis 移除键的时间，但是 expire 时间到了并不是实时触发移除的，会取决于过期键设置的删除策略：

> Expired (expired) events are generated when the Redis server deletes the key and not when the time to live theoretically reaches the value of zero.

而在监听的处理中，我们同样用这个 callback 做处理，并且在成功消费之后从补偿队列中移除对应的 key，代码就抽象为了：

```Go
func (w *Watcher) watchHandler(ctx context.Context, pubsub *redis.PubSub, callback Callback) {
    msg, err := pubsub.ReceiveMessage(ctx)
    if err != nil {
        fmt.Println("Error receiving message:", err)
    }
    splitPrefix := strings.Split(w.channel, ":")
    if len(splitPrefix) == 0 {
        fmt.Println("msg unknown:", msg)
        return
    }
    eventType := strings.TrimPrefix(msg.Channel, fmt.Sprintf("%s:", splitPrefix[0]))
    callback(EventType(eventType), msg.Payload)
    // 执行成功，则删除补偿队列中的 key
    w.redisClient.RemoveFromQueue(ctx, msg.Payload)
}
```

## Cluster Mode

在官方的 keyspace notifications 中会告诉你：

> Every node of a Redis cluster generates events about its own subset of the keyspace as described above. However, unlike regular Pub/Sub communication in a cluster, events' notifications are not broadcasted to all nodes. Put differently, keyspace events are node-specific. This means that to receive all keyspace events of a cluster, clients need to subscribe to each of the nodes.

简单总结就是尽管 Pub/Sub 其实是可以跨节点的，但是 keyspace notification 却不能，你需要监听每个节点：

```Go
func (r *RedisClusterClient) PSubscribe(ctx context.Context, channels ...string) []*redis.PubSub {
    var (
        pubsubs []*redis.PubSub
        mut     sync.Mutex
    )
    r.client.ForEachShard(ctx, func(ctx context.Context, client *redis.Client) error {
        // Function 是并发的，需要加锁
        mut.Lock()
        pubsubs = append(pubsubs, client.PSubscribe(ctx, channels...))
        mut.Unlock()
        return nil
    })
    return pubsubs
}
```

补充：cluster mode 也要记得 keyspace notification 需要是启用的，否则订阅不到，关于如何启用请参考 「Redis 配置」部分。

总结
完整代码见：https://github.com/csvwolf/go-redis-watcher-demo/tree/master

在实际的代码中还包括了池化手段，定时任务的简单实现，在本文中没有详细描述。

这里是一些实际生产中的建议：

- 如果你的实际处理函数是一些耗时任务，建议仍然加入可靠队列，比如 Redis 的 Stream 或者 MQ，再由队列进行分发和消费
- 请保证处理函数的设计是幂等的，避免重复执行造成的影响

尽管这里确实实现了一个相对可靠的实现，但是个人认为订阅删除键仍然是一个不够可靠的方法，可以有选择的使用（或者干脆算出时间塞 MQ 得了）。

（OS：真不知道公司是咋搞的保证可靠性的）
