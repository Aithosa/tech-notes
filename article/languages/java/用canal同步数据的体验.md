# 用 canal 同步数据的体验

部门的微服务会部署在三个独立环境中，这三个环境是完全隔离的，但是某个微服务有需求需要从环境 A 同步数据到 B 和 C，之前经过调研最后选择了 canal 来实现功能。

背景：

所设计的微服务会有一些业务资源配置，如机票的运价，航司仓位配置，航线公布运价等，这些确定 B 和 C 环境可以用和 A 环境一模一样的数据，并且业务平时主要活动在 A 环境，即我们给自己用的环境，B 和 C 是给其他客户部署的，不方便随时登录修改资源配置，而且也可以减小重复工作。

所涉及的 canal 同步机制和流程：
在 A 环境，canal 会读取 mysql 的 binlog，一旦检测到有数据变化，会读取 binlog 并解析内容，然后给 A 环境服务发送 Kafka，A 环境服务接收到 kafka 之后会把改动信息保存到数据库表 A 中，并且 A 环境设置一个定时任务，每隔设置的时间，就会把多少天之前的数据从表 A 移动到表 B 中(A 和 B 表结构相同，B 用于存储历史数据，其他环境也是如此)。

在 B 环境中(B 环境其实就不怎么涉及 canal 了)，有三个定时任务，定时任务 1 负责每 1 分钟从环境 A 的表 A 中拉取数据，数据必须不在 A 环境的表 A 和 B 中；定时任务 2 负责每两分钟读取表 A 中的未执行数据，把数据所涉及的改动在 B 环境也执行一次(按时间顺序取任务，先执行时间早的)，执行成功完会把数据从表 A 移动到表 B，并标记执行成功，执行失败的会留在表 A 中并标记失败，记录原因；定时任务 3 负责每隔设置的时间，把表 A 中执行失败的数据移动到表 B(也可以像 A 环境一样吧多少天之前的数据移动到表 B，这样更统一)。

如果有其他环境涉及从 A 环境同步数据，那么配置同 B 环境。

以上就是同步流程，实际上我们还有其他微服务和环境涉及使用 canal 同步数据，场景和这个不完全相同。能看出来 canal 同步不是那么完全可靠，如果按时间顺序一个改动一个改动同步(线性)，不出问题还好，出了问题前面同步失败了可能会导致后面同步也出问题，所以实现的时候需要尽量减小数据之间的耦合性，我们之前针对出现的问题做了优化，比如变更数据拉取重复，一次同步失败之后重复次数太多，改动执行时支持使用唯一索引等。

在讲述之前遇到过的问题时先谈下总体的使用感受吧，在同步数据时我们的目标当然是希望保证数据的完全一致性，涉及到的同步的表里大部分也都是确实需要达到这个目标的，当然也有少部分的表数据一致性可以没那么高，有几条同步失败了也没有什么影响，如公布运价(B 环境本身也有能力单挑刷新某航线的公布运价)。总体上要求数据同步一致性和可靠性没什么争议，并且这个同步场景而算是比较简单的。但是 canal 线性的同步比较容易导致前面失败了后面有几率出现数据不一致的情况，即使后来通过再次拉取数据又同步了一遍，这种打破线性同步的行为让我也难以确信同步完以后数据是否一定是一致的。订单服务虽然也设计 canal 同步，但是由于存在补偿机制，不开启 canal 同步也是可以的，而上面的同步就完全靠 canal。

之前遇到过的问题：

1. 一开始在 B 环境每条记录同步失败之后会自动重试 5 次，而一般同步失败一般是数据索引或者主键冲突这种，重试了也不会成功，所以浪费了大量资源；并且数据拉取没加锁，当数据拉取量非常大时一条记录会重复拉取多次，导致数据库和微服务负荷相当大；还有就是有的表没必要要求主键一定相同，只要主键之外的数据一致就行了，后来加了可以根据唯一索引同步。
2. 定时任务被关/删，这个情况遇到了好几次。之前存在定时任务被覆盖，要么被停了(微服务升级脚本的原因)，导致过了半个月，最长一次小半年才发现同步停了，这个时候就比较麻烦了，当然可以把 A 环境的同步记录重新拉到 B 环境重新执行，但是如上面所说，即使执行了，我对数据一致性有也没有那么大的信心，万一同步结果里还有几条失败的就更不敢确保一致性了，这时候就要重新用 A 环境的数据重置 B 环境，然后再次打开定时任务。
3. 还有就是 ICSL 修改了 Json 解析，导致 B 环境从 A 环境拉数据失败了，这个没啥好说的。
4. 数据同步失败没有告警，开发无权限查看生产数据库所以也不知道失败了，后期加了告警，可以订阅。
5. A 和 B 环境升级时间不一样，A 环境数据库表有变更，由于 B 环境要第二天升级，所以这段时间同步会失败，也会引发数据不一致的担心，后面做了表结构变更自动执行，也就是 A 环境变了，B 环境侦测到会自己该表结构(还没上线并且也不长该表，所以还不知道实际效果)。

总之如果生产环境平时没人乱动的话其实同步还算可靠，但是也没法一直做这个保证。
