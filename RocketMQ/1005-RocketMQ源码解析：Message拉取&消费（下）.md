# 1、概述

本文接：[《Message拉取&消费（上）》](https://github.com/YunaiV/Blog/blob/master/RocketMQ/1005-RocketMQ源码解析：Message拉取&消费（上）.md)。

主要解析 `Consumer` 在 **消费** 逻辑涉及到的源码。

# 2、Consumer

MQ 提供了两类消费者：

* PullConsumer：TODO
* PushConsumer：
    * 在大多数场景下使用。
    * 名字虽然是 `Push` 开头，实际在实现时，使用 `Pull` 方式实现。通过 `Pull` **不断不断不断**轮询 `Broker` 获取消息。当不存在新消息时，`Broker` 会**挂起请求**，直到有新消息产生，取消挂起，返回新消息。这样，基本和 `Broker` 主动 `Push` 做到**接近**的实时性（当然，还是有相应的实时性损失）。原理类似 **[长轮询( `Long-Polling` )](https://www.ibm.com/developerworks/cn/web/wa-lo-comet/)**。


**本文主要讲解`PushConsumer`，部分讲解`PullConsumer`，跳过`顺序消费`。**
**本文主要讲解`PushConsumer`，部分讲解`PullConsumer`，跳过`顺序消费`。**
**本文主要讲解`PushConsumer`，部分讲解`PullConsumer`，跳过`顺序消费`。**

# 3、PushConsumer 一览

先看一张 `PushConsumer` 包含的组件以及组件之间的交互图：

![PushConsumer手绘图.png](images/1005/PushConsumer手绘图.png)

* `RebalanceService`：负责分配消费队列，即分配当前 `Consumer` 可消费的队列。当有新的 `Consumer` 的加入或移除，都会进行消费队列重新负载均衡，分配消费队列。
* `PullMessageService`：拉取消息线程服务，**不断不断不断**从 `Broker` 拉取消息，并提交消费任务到 `ConsumeMessageService`。
* `ConsumeMessageService`：消费消息线程服务，**不断不断不断**消费消息，并处理消费结果。
* `RemoteBrokerOffsetStore`：`Consumer` 消费进度管理，负责从 `Broker` 获取消费进度，更新消费进度到 `Broker`。
* `ProcessQueue` ：消息处理队列。
* `MQClientInstance` ：封装对 `Namesrv`，`Broker` 的 API调用，提供给 `Producer`、`Consumer` 使用。

# 4、PushConsumer 消费队列分配

![RebalanceService&PushConsumer分配队列](images/1005/RebalanceService&PushConsumer分配队列.png)

## RebalanceService

```Java
  1: public class RebalanceService extends ServiceThread {
  2: 
  3:     /**
  4:      * 等待间隔，单位：毫秒
  5:      */
  6:     private static long waitInterval =
  7:         Long.parseLong(System.getProperty(
  8:             "rocketmq.client.rebalance.waitInterval", "20000"));
  9: 
 10:     private final Logger log = ClientLogger.getLog();
 11:     /**
 12:      * MQClient对象
 13:      */
 14:     private final MQClientInstance mqClientFactory;
 15: 
 16:     public RebalanceService(MQClientInstance mqClientFactory) {
 17:         this.mqClientFactory = mqClientFactory;
 18:     }
 19: 
 20:     @Override
 21:     public void run() {
 22:         log.info(this.getServiceName() + " service started");
 23: 
 24:         while (!this.isStopped()) {
 25:             this.waitForRunning(waitInterval);
 26:             this.mqClientFactory.doRebalance();
 27:         }
 28: 
 29:         log.info(this.getServiceName() + " service end");
 30:     }
 31: 
 32:     @Override
 33:     public String getServiceName() {
 34:         return RebalanceService.class.getSimpleName();
 35:     }
 36: }
```

* 说明 ：分配消费队列线程服务。
* 第 26 行 ：调用 `MQClientInstance#doRebalance(...)` 分配消费队列。目前有三种情况情况下触发：
    * 如 `第 25 行` 等待超时，每 20s 调用一次。
    * `PushConsumer` 启动时，调用 `rebalanceService#wakeup(...)` 触发。
    * `Broker` 通知 `Consumer` 加入 或 移除时，`Consumer` 响应通知，调用 `rebalanceService#wakeup(...)` 触发。

 详细解析见：[MQClientInstance#doRebalance(...)](mqclientinstancedorebalance)。

## MQClientInstance#doRebalance(...)

```Java
  1: public void doRebalance() {
  2:     for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
  3:         MQConsumerInner impl = entry.getValue();
  4:         if (impl != null) {
  5:             try {
  6:                 impl.doRebalance();
  7:             } catch (Throwable e) {
  8:                 log.error("doRebalance exception", e);
  9:             }
 10:         }
 11:     }
 12: }
```

* 说明 ：遍历当前 `Client` 包含的 `consumerTable`( `Consumer`集合 )，执行消费队列分配。
* **疑问**：目前代码调试下来，`consumerTable` 只包含 `Consumer` 自己。😈有大大对这个疑问有解答的，烦请解答下。
* 第 6 行 ：调用 `MQConsumerInner#doRebalance(...)` 进行队列分配。`DefaultMQPushConsumerImpl`、`DefaultMQPullConsumerImpl` 分别对该接口方法进行了实现。`DefaultMQPushConsumerImpl#doRebalance(...)` 详细解析见：[DefaultMQPushConsumerImpl#doRebalance(...)](defaultmqpushconsumerimpldorebalance)。

## DefaultMQPushConsumerImpl#doRebalance(...)

```Java
  1: public void doRebalance() {
  2:     if (!this.pause) {
  3:         this.rebalanceImpl.doRebalance(this.isConsumeOrderly());
  4:     }
  5: }
```

* 说明：执行消费队列分配。
* 第 3 行 ：调用 `RebalanceImpl#doRebalance(...)` 进行队列分配。详细解析见：[RebalancePushImpl#doRebalance(...)](rebalancepushimpldorebalance)。

## RebalanceImpl#doRebalance(...)

```Java
  1: /**
  2:  * 执行分配消费队列
  3:  *
  4:  * @param isOrder 是否顺序消息
  5:  */
  6: public void doRebalance(final boolean isOrder) {
  7:     // 分配每个 topic 的消息队列
  8:     Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
  9:     if (subTable != null) {
 10:         for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
 11:             final String topic = entry.getKey();
 12:             try {
 13:                 this.rebalanceByTopic(topic, isOrder);
 14:             } catch (Throwable e) {
 15:                 if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 16:                     log.warn("rebalanceByTopic Exception", e);
 17:                 }
 18:             }
 19:         }
 20:     }
 21:     // 移除未订阅的topic对应的消息队列
 22:     this.truncateMessageQueueNotMyTopic();
 23: }
 24: 
 25: /**
 26:  * 移除未订阅的消息队列
 27:  */
 28: private void truncateMessageQueueNotMyTopic() {
 29:     Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
 30:     for (MessageQueue mq : this.processQueueTable.keySet()) {
 31:         if (!subTable.containsKey(mq.getTopic())) {
 32: 
 33:             ProcessQueue pq = this.processQueueTable.remove(mq);
 34:             if (pq != null) {
 35:                 pq.setDropped(true);
 36:                 log.info("doRebalance, {}, truncateMessageQueueNotMyTopic remove unnecessary mq, {}", consumerGroup, mq);
 37:             }
 38:         }
 39:     }
 40: }
```

* `#doRebalance(...)` 说明 ：执行分配消费队列。
    * 第 7 至 20 行 ：循环订阅主题集合( `subscriptionInner` )，分配每一个 `Topic` 的消费队列。
    * 第 22 行 ：移除未订阅的 `Topic` 的消费队列。
* `#truncateMessageQueueNotMyTopic(...)` 说明 ：移除未订阅的消费队列。**当调用 `DefaultMQPushConsumer#unsubscribe(topic)` 时，只移除订阅主题集合( `subscriptionInner` )，对应消费队列移除在该方法。**

### RebalanceImpl#rebalanceByTopic(...)

```Java
  1: private void rebalanceByTopic(final String topic, final boolean isOrder) {
  2:     switch (messageModel) {
  3:         case BROADCASTING: {
  4:             Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
  5:             if (mqSet != null) {
  6:                 boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
  7:                 if (changed) {
  8:                     this.messageQueueChanged(topic, mqSet, mqSet);
  9:                     log.info("messageQueueChanged {} {} {} {}", //
 10:                         consumerGroup, //
 11:                         topic, //
 12:                         mqSet, //
 13:                         mqSet);
 14:                 }
 15:             } else {
 16:                 log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
 17:             }
 18:             break;
 19:         }
 20:         case CLUSTERING: {
 21:             // 获取 topic 对应的 队列 和 consumer信息
 22:             Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
 23:             List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
 24:             if (null == mqSet) {
 25:                 if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 26:                     log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
 27:                 }
 28:             }
 29: 
 30:             if (null == cidAll) {
 31:                 log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
 32:             }
 33: 
 34:             if (mqSet != null && cidAll != null) {
 35:                 // 排序 消费队列 和 消费者数组。因为是在Client进行分配队列，排序后，各Client的顺序才能保持一致。
 36:                 List<MessageQueue> mqAll = new ArrayList<>();
 37:                 mqAll.addAll(mqSet);
 38: 
 39:                 Collections.sort(mqAll);
 40:                 Collections.sort(cidAll);
 41: 
 42:                 AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;
 43: 
 44:                 // 根据 队列分配策略 分配消费队列
 45:                 List<MessageQueue> allocateResult;
 46:                 try {
 47:                     allocateResult = strategy.allocate(//
 48:                         this.consumerGroup, //
 49:                         this.mQClientFactory.getClientId(), //
 50:                         mqAll, //
 51:                         cidAll);
 52:                 } catch (Throwable e) {
 53:                     log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
 54:                         e);
 55:                     return;
 56:                 }
 57: 
 58:                 Set<MessageQueue> allocateResultSet = new HashSet<>();
 59:                 if (allocateResult != null) {
 60:                     allocateResultSet.addAll(allocateResult);
 61:                 }
 62: 
 63:                 // 更新消费队列
 64:                 boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
 65:                 if (changed) {
 66:                     log.info(
 67:                         "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
 68:                         strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
 69:                         allocateResultSet.size(), allocateResultSet);
 70:                     this.messageQueueChanged(topic, mqSet, allocateResultSet);
 71:                 }
 72:             }
 73:             break;
 74:         }
 75:         default:
 76:             break;
 77:     }
 78: }
 79: 
 80: /**
 81:  * 当负载均衡时，更新 消息处理队列
 82:  * - 移除 在processQueueTable && 不存在于 mqSet 里的消息队列
 83:  * - 增加 不在processQueueTable && 存在于mqSet 里的消息队列
 84:  *
 85:  * @param topic Topic
 86:  * @param mqSet 负载均衡结果后的消息队列数组
 87:  * @param isOrder 是否顺序
 88:  * @return 是否变更
 89:  */
 90: private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet, final boolean isOrder) {
 91:     boolean changed = false;
 92: 
 93:     // 移除 在processQueueTable && 不存在于 mqSet 里的消息队列
 94:     Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
 95:     while (it.hasNext()) { // TODO 待读：
 96:         Entry<MessageQueue, ProcessQueue> next = it.next();
 97:         MessageQueue mq = next.getKey();
 98:         ProcessQueue pq = next.getValue();
 99: 
100:         if (mq.getTopic().equals(topic)) {
101:             if (!mqSet.contains(mq)) { // 不包含的队列
102:                 pq.setDropped(true);
103:                 if (this.removeUnnecessaryMessageQueue(mq, pq)) {
104:                     it.remove();
105:                     changed = true;
106:                     log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
107:                 }
108:             } else if (pq.isPullExpired()) { // 队列拉取超时，进行清理
109:                 switch (this.consumeType()) {
110:                     case CONSUME_ACTIVELY:
111:                         break;
112:                     case CONSUME_PASSIVELY:
113:                         pq.setDropped(true);
114:                         if (this.removeUnnecessaryMessageQueue(mq, pq)) {
115:                             it.remove();
116:                             changed = true;
117:                             log.error("[BUG]doRebalance, {}, remove unnecessary mq, {}, because pull is pause, so try to fixed it",
118:                                 consumerGroup, mq);
119:                         }
120:                         break;
121:                     default:
122:                         break;
123:                 }
124:             }
125:         }
126:     }
127: 
128:     // 增加 不在processQueueTable && 存在于mqSet 里的消息队列。
129:     List<PullRequest> pullRequestList = new ArrayList<>(); // 拉消息请求数组
130:     for (MessageQueue mq : mqSet) {
131:         if (!this.processQueueTable.containsKey(mq)) {
132:             if (isOrder && !this.lock(mq)) {
133:                 log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
134:                 continue;
135:             }
136: 
137:             this.removeDirtyOffset(mq);
138:             ProcessQueue pq = new ProcessQueue();
139:             long nextOffset = this.computePullFromWhere(mq);
140:             if (nextOffset >= 0) {
141:                 ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
142:                 if (pre != null) {
143:                     log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
144:                 } else {
145:                     log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
146:                     PullRequest pullRequest = new PullRequest();
147:                     pullRequest.setConsumerGroup(consumerGroup);
148:                     pullRequest.setNextOffset(nextOffset);
149:                     pullRequest.setMessageQueue(mq);
150:                     pullRequest.setProcessQueue(pq);
151:                     pullRequestList.add(pullRequest);
152:                     changed = true;
153:                 }
154:             } else {
155:                 log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
156:             }
157:         }
158:     }
159: 
160:     // 发起消息拉取请求
161:     this.dispatchPullRequest(pullRequestList);
162: 
163:     return changed;
164: }
```

* `#rebalanceByTopic(...)` 说明 ：分配 `Topic` 的消费队列。
    * 第 3 至 19 行 ：广播模式( `BROADCASTING` ) 下，分配 `Topic` 对应的**所有**消费队列。   
    * 第 20 至 74 行 ：集群模式( `CLUSTERING` ) 下，分配 `Topic` 对应的**部分**消费队列。
        * 第 21 至 40 行 ：获取 `Topic` 对应的队列和消费者们，并对其进行排序。因为各 `Consumer` 是在本地分配消费队列，排序后才能保证各 `Consumer` 顺序一致。
        *  第 42 至 61 行 ：根据 队列分配策略( `AllocateMessageQueueStrategy` ) 分配消费队列。详细解析见：[AllocateMessageQueueStrategy](#allocatemessagequeuestrategy)。
        *  第 63 至 72 行 ：更新 `Topic` 对应的消费队列。
* `#updateProcessQueueTableInRebalance(...)` 说明 ：当分配队列时，更新 `Topic` 对应的消费队列，并返回是否有变更。
    * 第 93 至 126 行 ：移除不存在于 分配的消费队列( `mqSet` ) 的 消息处理队列( `processQueueTable` )。
        * 第 103 行 ：移除不需要的消费队列。详细解析见：[RebalancePushImpl#removeUnnecessaryMessageQueue(...)](#rebalancepushimplremoveunnecessarymessagequeue)。
        * 第 108 行 ：队列拉取超时，即 `当前时间 - 最后一次拉取消息时间 > 120s` ( 120s 可配置)，判定发生 **BUG**，过久未进行消息拉取，移除队列。移除后，下面**#新增队列逻辑#**可以重新加入新的该队列。
    * 第 128 至 158 行 ：增加 分配的消费队列( `mqSet` ) 新增的消费队列。
        * 第 132 至 135 行 ：`顺序消费` 相关跳过，详细解析见：[《Message顺序发送与消费》](https://github.com/YunaiV/Blog/blob/master/RocketMQ/1007-RocketMQ源码解析：Message顺序发送与消费.md)。
        * 第 137 行 ：移除队列的消费进度。
        * 第 139 行 ：获取队列消费进度。详细解析见：[RebalancePushImpl#computePullFromWhere(...)](#rebalancepushimplcomputepullfromwhere)。
        * 第 140 至 156 行 ：**添加新消费处理队列，添加消费拉取消息请求**。
    * 第 161 行 ：**发起新增的消费队列消息拉取请求**。详细解析见：TOTOTOTO

### RebalanceImpl#removeUnnecessaryMessageQueue(...)

#### RebalancePushImpl#removeUnnecessaryMessageQueue(...)

```Java
  1: public boolean removeUnnecessaryMessageQueue(MessageQueue mq, ProcessQueue pq) {
  2:     // 同步队列的消费进度，并移除之。
  3:     this.defaultMQPushConsumerImpl.getOffsetStore().persist(mq);
  4:     this.defaultMQPushConsumerImpl.getOffsetStore().removeOffset(mq);
  5:     // TODO 顺序消费
  6:     if (this.defaultMQPushConsumerImpl.isConsumeOrderly()
  7:         && MessageModel.CLUSTERING.equals(this.defaultMQPushConsumerImpl.messageModel())) {
  8:         try {
  9:             if (pq.getLockConsume().tryLock(1000, TimeUnit.MILLISECONDS)) {
 10:                 try {
 11:                     return this.unlockDelay(mq, pq);
 12:                 } finally {
 13:                     pq.getLockConsume().unlock();
 14:                 }
 15:             } else {
 16:                 log.warn("[WRONG]mq is consuming, so can not unlock it, {}. maybe hanged for a while, {}", //
 17:                     mq, //
 18:                     pq.getTryUnlockTimes());
 19: 
 20:                 pq.incTryUnlockTimes();
 21:             }
 22:         } catch (Exception e) {
 23:             log.error("removeUnnecessaryMessageQueue Exception", e);
 24:         }
 25: 
 26:         return false;
 27:     }
 28:     return true;
 29: }
```

* 说明 ：移除不需要的消费队列相关的信息，并返回是否移除成功。
* 第 2 至 4 行 ：**同步**队列的消费进度，并移除之。
* 第 5 至 27 行 ：`顺序消费` 相关跳过，详细解析见：[《Message顺序发送与消费》](https://github.com/YunaiV/Blog/blob/master/RocketMQ/1007-RocketMQ源码解析：Message顺序发送与消费.md)。

#### `[PullConsumer]` RebalancePullImpl#removeUnnecessaryMessageQueue(...)

```Java
  1: public boolean removeUnnecessaryMessageQueue(MessageQueue mq, ProcessQueue pq) {
  2:     this.defaultMQPullConsumerImpl.getOffsetStore().persist(mq);
  3:     this.defaultMQPullConsumerImpl.getOffsetStore().removeOffset(mq);
  4:     return true;
  5: }
```

* 说明 ：移除不需要的消费队列相关的信息，并返回移除成功。**和`RebalancePushImpl#removeUnnecessaryMessageQueue(...)`基本一致。**

### AllocateMessageQueueStrategy

![AllocateMessageQueueStrategy类图](images/1005/AllocateMessageQueueStrategy类图.png)

#### AllocateMessageQueueAveragely

```Java
  1: public class AllocateMessageQueueAveragely implements AllocateMessageQueueStrategy {
  2:     private final Logger log = ClientLogger.getLog();
  3: 
  4:     @Override
  5:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  6:         List<String> cidAll) {
  7:         // 校验参数是否正确
  8:         if (currentCID == null || currentCID.length() < 1) {
  9:             throw new IllegalArgumentException("currentCID is empty");
 10:         }
 11:         if (mqAll == null || mqAll.isEmpty()) {
 12:             throw new IllegalArgumentException("mqAll is null or mqAll empty");
 13:         }
 14:         if (cidAll == null || cidAll.isEmpty()) {
 15:             throw new IllegalArgumentException("cidAll is null or cidAll empty");
 16:         }
 17: 
 18:         List<MessageQueue> result = new ArrayList<>();
 19:         if (!cidAll.contains(currentCID)) {
 20:             log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
 21:                 consumerGroup,
 22:                 currentCID,
 23:                 cidAll);
 24:             return result;
 25:         }
 26:         // 平均分配
 27:         int index = cidAll.indexOf(currentCID); // 第几个consumer。
 28:         int mod = mqAll.size() % cidAll.size(); // 余数，即多少消息队列无法平均分配。
 29:         int averageSize =
 30:             mqAll.size() <= cidAll.size() ? 1 : (mod > 0 && index < mod ? mqAll.size() / cidAll.size()
 31:                 + 1 : mqAll.size() / cidAll.size());
 32:         int startIndex = (mod > 0 && index < mod) ? index * averageSize : index * averageSize + mod; // 有余数的情况下，[0, mod) 平分余数，即每consumer多分配一个节点；第index开始，跳过前mod余数。
 33:         int range = Math.min(averageSize, mqAll.size() - startIndex); // 分配队列数量。之所以要Math.min()的原因是，mqAll.size() <= cidAll.size()，部分consumer分配不到消费队列。
 34:         for (int i = 0; i < range; i++) {
 35:             result.add(mqAll.get((startIndex + i) % mqAll.size()));
 36:         }
 37:         return result;
 38:     }
 39: 
 40:     @Override
 41:     public String getName() {
 42:         return "AVG";
 43:     }
 44: }
```

* 说明 ：**平均**分配队列策略。
* 第 7 至 25 行 ：参数校验。
* 第 26 至 36 行 ：平均分配消费队列。
    * 第 27 行 ：`index` ：当前 `Consumer` 在消费集群里是第几个。这里就是为什么需要对传入的 `cidAll` 参数必须进行排序的原因。如果不排序，`Consumer` 在本地计算出来的 `index` 无法一致，影响计算结果。
    * 第 28 行 ：`mod` ：余数，即多少消费队列无法平均分配。
    * 第 29 至 31 行 ：`averageSize` ：代码可以简化成 `(mod > 0 && index < mod ? mqAll.size() / cidAll.size() + 1 : mqAll.size() / cidAll.size())`。
        * `[ 0, mod )` ：`mqAll.size() / cidAll.size() + 1`。前面 `mod` 个 `Consumer` 平分余数，多获得 1 个消费队列。
        * `[ mod, cidAll.size() )` ：`mqAll.size() / cidAll.size()`。
    * 第 32 行 ：`startIndex` ：`Consumer` 分配消息队列开始位置。
    * 第 33 行 ：`range` ：分配队列数量。之所以要 `Math#min(...)` 的原因：当 `mqAll.size() <= cidAll.size()` 时，最后几个 `Consumer` 分配不到消费队列。
    * 第 34 至 36 行 ：生成分配消费队列结果。
* 举个例子：

固定消费队列长度为**4**。

|   | Consumer * 2 *可以整除* | Consumer * 3 *不可整除* | Consumer * 5 *无法都分配* |
| --- | --- | --- | --- |
| 消费队列[0] | Consumer[0] | Consumer[0] | Consumer[0] |
| 消费队列[1] | Consumer[0] | Consumer[0] | Consumer[1] |
| 消费队列[2] | Consumer[1] | Consumer[1] | Consumer[2] |
| 消费队列[3] | Consumer[1] | Consumer[2] | Consumer[3] |

#### AllocateMessageQueueByMachineRoom

```Java
  1: public class AllocateMessageQueueByMachineRoom implements AllocateMessageQueueStrategy {
  2:     /**
  3:      * 消费者消费brokerName集合
  4:      */
  5:     private Set<String> consumeridcs;
  6: 
  7:     @Override
  8:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  9:         List<String> cidAll) {
 10:         // 参数校验
 11:         List<MessageQueue> result = new ArrayList<MessageQueue>();
 12:         int currentIndex = cidAll.indexOf(currentCID);
 13:         if (currentIndex < 0) {
 14:             return result;
 15:         }
 16:         // 计算符合当前配置的消费者数组('consumeridcs')对应的消费队列
 17:         List<MessageQueue> premqAll = new ArrayList<MessageQueue>();
 18:         for (MessageQueue mq : mqAll) {
 19:             String[] temp = mq.getBrokerName().split("@");
 20:             if (temp.length == 2 && consumeridcs.contains(temp[0])) {
 21:                 premqAll.add(mq);
 22:             }
 23:         }
 24:         // 平均分配
 25:         int mod = premqAll.size() / cidAll.size();
 26:         int rem = premqAll.size() % cidAll.size();
 27:         int startIndex = mod * currentIndex;
 28:         int endIndex = startIndex + mod;
 29:         for (int i = startIndex; i < endIndex; i++) {
 30:             result.add(mqAll.get(i));
 31:         }
 32:         if (rem > currentIndex) {
 33:             result.add(premqAll.get(currentIndex + mod * cidAll.size()));
 34:         }
 35:         return result;
 36:     }
 37: 
 38:     @Override
 39:     public String getName() {
 40:         return "MACHINE_ROOM";
 41:     }
 42: 
 43:     public Set<String> getConsumeridcs() {
 44:         return consumeridcs;
 45:     }
 46: 
 47:     public void setConsumeridcs(Set<String> consumeridcs) {
 48:         this.consumeridcs = consumeridcs;
 49:     }
 50: }
```

* 说明 ：**平均**分配**可消费的** `Broker` 对应的消费队列。
* 第 7 至 15 行 ：参数校验。
* 第 16 至 23 行 ：计算**可消费的** `Broker` 对应的消费队列。
* 第 25 至 34 行 ：平均分配消费队列。该**平均分配**方式和 `AllocateMessageQueueAveragely` 略有不同，其是将多余的结尾部分分配给前 `rem` 个 `Consumer`。
* 疑问：*比较疑惑使用该分配策略，`Consumer` 和 `Broker` 分配需要怎么配置*。😈等研究**主从**相关源码时，仔细考虑下。

#### AllocateMessageQueueAveragelyByCircle

 ```Java
   1: public class AllocateMessageQueueAveragelyByCircle implements AllocateMessageQueueStrategy {
  2:     private final Logger log = ClientLogger.getLog();
  3: 
  4:     @Override
  5:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  6:         List<String> cidAll) {
  7:         // 校验参数是否正确
  8:         if (currentCID == null || currentCID.length() < 1) {
  9:             throw new IllegalArgumentException("currentCID is empty");
 10:         }
 11:         if (mqAll == null || mqAll.isEmpty()) {
 12:             throw new IllegalArgumentException("mqAll is null or mqAll empty");
 13:         }
 14:         if (cidAll == null || cidAll.isEmpty()) {
 15:             throw new IllegalArgumentException("cidAll is null or cidAll empty");
 16:         }
 17: 
 18:         List<MessageQueue> result = new ArrayList<MessageQueue>();
 19:         if (!cidAll.contains(currentCID)) {
 20:             log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
 21:                 consumerGroup,
 22:                 currentCID,
 23:                 cidAll);
 24:             return result;
 25:         }
 26: 
 27:         // 环状分配
 28:         int index = cidAll.indexOf(currentCID);
 29:         for (int i = index; i < mqAll.size(); i++) {
 30:             if (i % cidAll.size() == index) {
 31:                 result.add(mqAll.get(i));
 32:             }
 33:         }
 34:         return result;
 35:     }
 36: 
 37:     @Override
 38:     public String getName() {
 39:         return "AVG_BY_CIRCLE";
 40:     }
 41: }
 ```
 
 * 说明 ：环状分配消费队列。

#### AllocateMessageQueueByConfig

```Java
  1: public class AllocateMessageQueueByConfig implements AllocateMessageQueueStrategy {
  2:     private List<MessageQueue> messageQueueList;
  3: 
  4:     @Override
  5:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  6:         List<String> cidAll) {
  7:         return this.messageQueueList;
  8:     }
  9: 
 10:     @Override
 11:     public String getName() {
 12:         return "CONFIG";
 13:     }
 14: 
 15:     public List<MessageQueue> getMessageQueueList() {
 16:         return messageQueueList;
 17:     }
 18: 
 19:     public void setMessageQueueList(List<MessageQueue> messageQueueList) {
 20:         this.messageQueueList = messageQueueList;
 21:     }
 22: }
```

* 说明 ：分配配置的消息队列。
* 疑问 ：*疑惑该分配策略的使用场景。*

# 5、PushConsumer 消费进度读取

## RebalancePushImpl#computePullFromWhere(...)

## `[PullConsumer]` RebalancePullImpl#computePullFromWhere(...)

# 7、Consumer 调用[拉取消息]接口
# 8、Consumer 消费消息
# 9、Consumer 调用[发回消息]接口
# 10、Consumer 调用[更新消费进度]接口


