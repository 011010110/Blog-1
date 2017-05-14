# 1. 概述

本文主要解析 `Namesrv`、`Broker` 如何实现高可用，`Producer`、`Consumer` 怎么与它们通信保证高可用。

# 2. Namesrv 高可用

**启动多个 `Namesrv` 实现高可用。**  
相较于 `Zookeeper`、`Consul`、`Etcd` 等，`Namesrv` 是一个**超轻量级**的注册中心，提供**命名服务**。

## 2.1 Broker 注册到 Namesrv

* 📌 **多个 `Namesrv` 之间，没有任何关系（不存在类似 `Zookeeper` 的 `Leader`/`Follower` 等角色），不进行通信与数据同步。通过 `Broker` 循环注册多个 `Namesrv`。**

```Java
  1: // ⬇️⬇️⬇️【NettyRemotingClient.java】
  2: public RegisterBrokerResult registerBrokerAll(
  3:     final String clusterName,
  4:     final String brokerAddr,
  5:     final String brokerName,
  6:     final long brokerId,
  7:     final String haServerAddr,
  8:     final TopicConfigSerializeWrapper topicConfigWrapper,
  9:     final List<String> filterServerList,
 10:     final boolean oneway,
 11:     final int timeoutMills) {
 12:     RegisterBrokerResult registerBrokerResult = null;
 13: 
 14:     List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
 15:     if (nameServerAddressList != null) {
 16:         for (String namesrvAddr : nameServerAddressList) { // 循环多个 Namesrv
 17:             try {
 18:                 RegisterBrokerResult result = this.registerBroker(namesrvAddr, clusterName, brokerAddr, brokerName, brokerId,
 19:                     haServerAddr, topicConfigWrapper, filterServerList, oneway, timeoutMills);
 20:                 if (result != null) {
 21:                     registerBrokerResult = result;
 22:                 }
 23: 
 24:                 log.info("register broker to name server {} OK", namesrvAddr);
 25:             } catch (Exception e) {
 26:                 log.warn("registerBroker Exception, {}", namesrvAddr, e);
 27:             }
 28:         }
 29:     }
 30: 
 31:     return registerBrokerResult;
 32: }
```

## 2.2 Producer、Consumer 访问 Namesrv

* 📌 **`Producer`、`Consumer` 从 `Namesrv`列表选择一个可连接的进行通信。**

```Java
  1: // ⬇️⬇️⬇️【NettyRemotingClient.java】
  2: private Channel getAndCreateNameserverChannel() throws InterruptedException {
  3:     // 返回已选择、可连接Namesrv
  4:     String addr = this.namesrvAddrChoosed.get();
  5:     if (addr != null) {
  6:         ChannelWrapper cw = this.channelTables.get(addr);
  7:         if (cw != null && cw.isOK()) {
  8:             return cw.getChannel();
  9:         }
 10:     }
 11:     //
 12:     final List<String> addrList = this.namesrvAddrList.get();
 13:     if (this.lockNamesrvChannel.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
 14:         try {
 15:             // 返回已选择、可连接的Namesrv
 16:             addr = this.namesrvAddrChoosed.get();
 17:             if (addr != null) {
 18:                 ChannelWrapper cw = this.channelTables.get(addr);
 19:                 if (cw != null && cw.isOK()) {
 20:                     return cw.getChannel();
 21:                 }
 22:             }
 23:             // 从【Namesrv列表】中选择一个连接的返回
 24:             if (addrList != null && !addrList.isEmpty()) {
 25:                 for (int i = 0; i < addrList.size(); i++) {
 26:                     int index = this.namesrvIndex.incrementAndGet();
 27:                     index = Math.abs(index);
 28:                     index = index % addrList.size();
 29:                     String newAddr = addrList.get(index);
 30: 
 31:                     this.namesrvAddrChoosed.set(newAddr);
 32:                     Channel channelNew = this.createChannel(newAddr);
 33:                     if (channelNew != null)
 34:                         return channelNew;
 35:                 }
 36:             }
 37:         } catch (Exception e) {
 38:             log.error("getAndCreateNameserverChannel: create name server channel exception", e);
 39:         } finally {
 40:             this.lockNamesrvChannel.unlock();
 41:         }
 42:     } else {
 43:         log.warn("getAndCreateNameserverChannel: try to lock name server, but timeout, {}ms", LOCK_TIMEOUT_MILLIS);
 44:     }
 45: 
 46:     return null;
 47: }
```

# 3. Broker 高可用

**启动多个 `Broker集群` 实现高可用。**  
**`Broker集群` = `Master节点`x1 + `Slave节点`xN。**  
类似 `MySQL`，`Master节点` 提供**读写**服务，`Slave节点` 只提供**读**服务。  

## 3.1 Broker 主从

* **每个集群，`Slave`节点 从 `Master`节点 不断拉取 `CommitLog`。**
* **集群 与 集群 之间没有任何关系，不进行通信与数据同步。**

集群内，`Master`节点 有**两种**类型：`Master_Sync`、`Master_Async`：前者在 `Producer` 发送消息时，等待 `Slave`节点 存储完毕后再返回发送结果，而后者不需要等待。

-------

### 3.1.1 组件

再看具体实现代码之前，我们来看看 `Master`/`Slave`节点 包含的组件：  
![HA组件图.png](images/1009/HA组件图.png)

* `Master`节点
    * `AcceptSocketService` ：接收 `Slave`节点 连接。
    * `HAConnection`
        * `ReadSocketService` ：**读**来自 `Slave`节点 的数据。 
        * `WriteSocketService` ：**写**到往 `Slave`节点 的数据。
* `Slave`节点
    * `HAClient` ：对 `Master`节点 连接、读写数据。

### 3.1.2 通信协议

`Master`节点 与 `Slave`节点 **通信协议**很简单，只有如下两条。

| 对象 | 用途 | 第几位 | 字段 | 数据类型 | 字节数 | 说明
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Slave=>Master | 上报CommitLog最大物理位置 |  |  |  |  |  |
|  | | 0 | maxPhyOffset  |  Long | 8 | CommitLog最大物理位置 |
| Master=>Slave | 传递CommitLog内容 |  |  |  |  |  |
| | | 0 | fromPhyOffset | Long | 8 | CommitLog开始物理位置 | 
| | | 1 | size | Int | 4 | 同步CommitLog内容长度 | 
| | | 2 | body | Bytes | size | 同步CommitLog内容 | 

### 3.1.3 Slave

![HAClient顺序图](images/1009/HAClient顺序图.png)

```Java
  1: // ⬇️⬇️⬇️【HAClient.java】
  2: public void run() {
  3:     log.info(this.getServiceName() + " service started");
  4: 
  5:     while (!this.isStopped()) {
  6:         try {
  7:             if (this.connectMaster()) {
  8:                 // 若到满足上报间隔，上报到Master进度
  9:                 if (this.isTimeToReportOffset()) {
 10:                     boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
 11:                     if (!result) {
 12:                         this.closeMaster();
 13:                     }
 14:                 }
 15: 
 16:                 this.selector.select(1000);
 17: 
 18:                 // 处理读取事件
 19:                 boolean ok = this.processReadEvent();
 20:                 if (!ok) {
 21:                     this.closeMaster();
 22:                 }
 23: 
 24:                 // 若进度有变化，上报到Master进度
 25:                 if (!reportSlaveMaxOffsetPlus()) {
 26:                     continue;
 27:                 }
 28: 
 29:                 // Master过久未返回数据，关闭连接
 30:                 long interval = HAService.this.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;
 31:                 if (interval > HAService.this.getDefaultMessageStore().getMessageStoreConfig()
 32:                     .getHaHousekeepingInterval()) {
 33:                     log.warn("HAClient, housekeeping, found this connection[" + this.masterAddress
 34:                         + "] expired, " + interval);
 35:                     this.closeMaster();
 36:                     log.warn("HAClient, master not response some time, so close connection");
 37:                 }
 38:             } else {
 39:                 this.waitForRunning(1000 * 5);
 40:             }
 41:         } catch (Exception e) {
 42:             log.warn(this.getServiceName() + " service has exception. ", e);
 43:             this.waitForRunning(1000 * 5);
 44:         }
 45:     }
 46: 
 47:     log.info(this.getServiceName() + " service end");
 48: }
```
* ⬆️⬆️⬆️
* 说明 ：`Slave` 主循环，实现了**不断不断不断**从 `Master` 读取 `CommitLog` 内容。
* 第 8 至 14 行 ：**固定间隔（默认5s）**向 `Master` 上报 `Slave` 本地 `CommitLog` 最大物理位置。该操作有两个作用：（1）`Slave` 向 `Master` 拉取 `CommitLog` 内容请求；（2）心跳。
* 第 16 至 22 行 ：处理 `Master` 发来 `Slave` 的 `CommitLog` 内容。

### 3.1.4 Master

## 3.2 Producer 发送消息

## 3.3 Consumer 消费消息

// TODO 从节点消费

