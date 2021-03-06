-------

![](http://www.yunai.me/images/common/wechat_mp.jpeg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。

-------


- [1. 概述](#)
- [2. 事务消息发送](#)
	- [2.1 Producer 发送事务消息](#)
	- [2.2 Broker 处理结束事务请求](#)
	- [2.3 Broker 生成 ConsumeQueue](#)
- [3. 事务消息回查](#)
	- [3.1 Broker 发起【事务消息回查】](#)
		- [3.1.1 官方V3.1.4：基于文件系统](#)
			- [3.1.1.1 存储消息到 CommitLog](#)
			- [3.1.1.2 写【事务消息】状态存储（TranStateTable）](#)
			- [3.1.1.3 【事务消息】回查](#)
			- [3.1.1.4 初始化【事务消息】状态存储（TranStateTable）](#)
			- [3.1.1.5 补充](#)
		- [3.1.2 官方V4.0.0：基于数据库](#)
	- [3.2 Producer 接收【事务消息回查】](#)

# 1. 概述

**必须必须必须** 前置阅读内容：

* [《事务消息（阿里云）》](https://help.aliyun.com/document_detail/43348.html?spm=5176.doc43490.6.566.Zd5Bl7)

# 2. 事务消息发送

## 2.1 Producer 发送事务消息

* 活动图如下（结合 `核心代码` 理解）：

![Producer发送事务消息](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1011/Producer发送事务消息.png)

* 实现代码如下：

```Java
  1: // ⬇️⬇️⬇️【DefaultMQProducerImpl.java】
  2: /**
  3:  * 发送事务消息
  4:  *
  5:  * @param msg 消息
  6:  * @param tranExecuter 【本地事务】执行器
  7:  * @param arg 【本地事务】执行器参数
  8:  * @return 事务发送结果
  9:  * @throws MQClientException 当 Client 发生异常时
 10:  */
 11: public TransactionSendResult sendMessageInTransaction(final Message msg, final LocalTransactionExecuter tranExecuter, final Object arg)
 12:     throws MQClientException {
 13:     if (null == tranExecuter) {
 14:         throw new MQClientException("tranExecutor is null", null);
 15:     }
 16:     Validators.checkMessage(msg, this.defaultMQProducer);
 17: 
 18:     // 发送【Half消息】
 19:     SendResult sendResult;
 20:     MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
 21:     MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
 22:     try {
 23:         sendResult = this.send(msg);
 24:     } catch (Exception e) {
 25:         throw new MQClientException("send message Exception", e);
 26:     }
 27: 
 28:     // 处理发送【Half消息】结果
 29:     LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
 30:     Throwable localException = null;
 31:     switch (sendResult.getSendStatus()) {
 32:         // 发送【Half消息】成功，执行【本地事务】逻辑
 33:         case SEND_OK: {
 34:             try {
 35:                 if (sendResult.getTransactionId() != null) { // 事务编号。目前开源版本暂时没用到，猜想ONS在使用。
 36:                     msg.putUserProperty("__transactionId__", sendResult.getTransactionId());
 37:                 }
 38: 
 39:                 // 执行【本地事务】逻辑
 40:                 localTransactionState = tranExecuter.executeLocalTransactionBranch(msg, arg);
 41:                 if (null == localTransactionState) {
 42:                     localTransactionState = LocalTransactionState.UNKNOW;
 43:                 }
 44: 
 45:                 if (localTransactionState != LocalTransactionState.COMMIT_MESSAGE) {
 46:                     log.info("executeLocalTransactionBranch return {}", localTransactionState);
 47:                     log.info(msg.toString());
 48:                 }
 49:             } catch (Throwable e) {
 50:                 log.info("executeLocalTransactionBranch exception", e);
 51:                 log.info(msg.toString());
 52:                 localException = e;
 53:             }
 54:         }
 55:         break;
 56:         // 发送【Half消息】失败，标记【本地事务】状态为回滚
 57:         case FLUSH_DISK_TIMEOUT:
 58:         case FLUSH_SLAVE_TIMEOUT:
 59:         case SLAVE_NOT_AVAILABLE:
 60:             localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
 61:             break;
 62:         default:
 63:             break;
 64:     }
 65: 
 66:     // 结束事务：提交消息 COMMIT / ROLLBACK
 67:     try {
 68:         this.endTransaction(sendResult, localTransactionState, localException);
 69:     } catch (Exception e) {
 70:         log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);
 71:     }
 72: 
 73:     // 返回【事务发送结果】
 74:     TransactionSendResult transactionSendResult = new TransactionSendResult();
 75:     transactionSendResult.setSendStatus(sendResult.getSendStatus());
 76:     transactionSendResult.setMessageQueue(sendResult.getMessageQueue());
 77:     transactionSendResult.setMsgId(sendResult.getMsgId());
 78:     transactionSendResult.setQueueOffset(sendResult.getQueueOffset());
 79:     transactionSendResult.setTransactionId(sendResult.getTransactionId());
 80:     transactionSendResult.setLocalTransactionState(localTransactionState);
 81:     return transactionSendResult;
 82: }
 83: 
 84: /**
 85:  * 结束事务：提交消息 COMMIT / ROLLBACK
 86:  *
 87:  * @param sendResult 发送【Half消息】结果
 88:  * @param localTransactionState 【本地事务】状态
 89:  * @param localException 执行【本地事务】逻辑产生的异常
 90:  * @throws RemotingException 当远程调用发生异常时
 91:  * @throws MQBrokerException 当 Broker 发生异常时
 92:  * @throws InterruptedException 当线程中断时
 93:  * @throws UnknownHostException 当解码消息编号失败是
 94:  */
 95: public void endTransaction(//
 96:     final SendResult sendResult, //
 97:     final LocalTransactionState localTransactionState, //
 98:     final Throwable localException) throws RemotingException, MQBrokerException, InterruptedException, UnknownHostException {
 99:     // 解码消息编号
100:     final MessageId id;
101:     if (sendResult.getOffsetMsgId() != null) {
102:         id = MessageDecoder.decodeMessageId(sendResult.getOffsetMsgId());
103:     } else {
104:         id = MessageDecoder.decodeMessageId(sendResult.getMsgId());
105:     }
106: 
107:     // 创建请求
108:     String transactionId = sendResult.getTransactionId();
109:     final String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(sendResult.getMessageQueue().getBrokerName());
110:     EndTransactionRequestHeader requestHeader = new EndTransactionRequestHeader();
111:     requestHeader.setTransactionId(transactionId);
112:     requestHeader.setCommitLogOffset(id.getOffset());
113:     switch (localTransactionState) {
114:         case COMMIT_MESSAGE:
115:             requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
116:             break;
117:         case ROLLBACK_MESSAGE:
118:             requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
119:             break;
120:         case UNKNOW:
121:             requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
122:             break;
123:         default:
124:             break;
125:     }
126:     requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
127:     requestHeader.setTranStateTableOffset(sendResult.getQueueOffset());
128:     requestHeader.setMsgId(sendResult.getMsgId());
129:     String remark = localException != null ? ("executeLocalTransactionBranch exception: " + localException.toString()) : null;
130: 
131:     // 提交消息 COMMIT / ROLLBACK。！！！通信方式为：Oneway！！！
132:     this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, requestHeader, remark, this.defaultMQProducer.getSendMsgTimeout());
133: }
```

## 2.2 Broker 处理结束事务请求

* 🦅 查询请求的消息，进行**提交 / 回滚**。实现代码如下：

```Java
  1: // ⬇️⬇️⬇️【EndTransactionProcessor.java】
  2: public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
  3:     final RemotingCommand response = RemotingCommand.createResponseCommand(null);
  4:     final EndTransactionRequestHeader requestHeader = (EndTransactionRequestHeader) request.decodeCommandCustomHeader(EndTransactionRequestHeader.class);
  5: 
  6:     // 省略代码 =》打印日志（只处理 COMMIT / ROLLBACK）
  7: 
  8:     // 查询提交的消息
  9:     final MessageExt msgExt = this.brokerController.getMessageStore().lookMessageByOffset(requestHeader.getCommitLogOffset());
 10:     if (msgExt != null) {
 11:         // 省略代码 =》校验消息
 12: 
 13:         // 生成消息
 14:         MessageExtBrokerInner msgInner = this.endMessageTransaction(msgExt);
 15:         msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), requestHeader.getCommitOrRollback()));
 16:         msgInner.setQueueOffset(requestHeader.getTranStateTableOffset());
 17:         msgInner.setPreparedTransactionOffset(requestHeader.getCommitLogOffset());
 18:         msgInner.setStoreTimestamp(msgExt.getStoreTimestamp());
 19:         if (MessageSysFlag.TRANSACTION_ROLLBACK_TYPE == requestHeader.getCommitOrRollback()) {
 20:             msgInner.setBody(null);
 21:         }
 22: 
 23:         // 存储生成消息
 24:         final MessageStore messageStore = this.brokerController.getMessageStore();
 25:         final PutMessageResult putMessageResult = messageStore.putMessage(msgInner);
 26: 
 27:         // 处理存储结果
 28:         if (putMessageResult != null) {
 29:             switch (putMessageResult.getPutMessageStatus()) {
 30:                 // Success
 31:                 case PUT_OK:
 32:                 case FLUSH_DISK_TIMEOUT:
 33:                 case FLUSH_SLAVE_TIMEOUT:
 34:                 case SLAVE_NOT_AVAILABLE:
 35:                     response.setCode(ResponseCode.SUCCESS);
 36:                     response.setRemark(null);
 37:                     break;
 38:                 // Failed
 39:                 case CREATE_MAPEDFILE_FAILED:
 40:                     response.setCode(ResponseCode.SYSTEM_ERROR);
 41:                     response.setRemark("create maped file failed.");
 42:                     break;
 43:                 case MESSAGE_ILLEGAL:
 44:                 case PROPERTIES_SIZE_EXCEEDED:
 45:                     response.setCode(ResponseCode.MESSAGE_ILLEGAL);
 46:                     response.setRemark("the message is illegal, maybe msg body or properties length not matched. msg body length limit 128k, msg properties length limit 32k.");
 47:                     break;
 48:                 case SERVICE_NOT_AVAILABLE:
 49:                     response.setCode(ResponseCode.SERVICE_NOT_AVAILABLE);
 50:                     response.setRemark("service not available now.");
 51:                     break;
 52:                 case OS_PAGECACHE_BUSY:
 53:                     response.setCode(ResponseCode.SYSTEM_ERROR);
 54:                     response.setRemark("OS page cache busy, please try another machine");
 55:                     break;
 56:                 case UNKNOWN_ERROR:
 57:                     response.setCode(ResponseCode.SYSTEM_ERROR);
 58:                     response.setRemark("UNKNOWN_ERROR");
 59:                     break;
 60:                 default:
 61:                     response.setCode(ResponseCode.SYSTEM_ERROR);
 62:                     response.setRemark("UNKNOWN_ERROR DEFAULT");
 63:                     break;
 64:             }
 65: 
 66:             return response;
 67:         } else {
 68:             response.setCode(ResponseCode.SYSTEM_ERROR);
 69:             response.setRemark("store putMessage return null");
 70:         }
 71:     } else {
 72:         response.setCode(ResponseCode.SYSTEM_ERROR);
 73:         response.setRemark("find prepared transaction message failed");
 74:         return response;
 75:     }
 76: 
 77:     return response;
 78: }
```

## 2.3 Broker 生成 ConsumeQueue

* 🦅 事务消息，提交（`COMMIT`）后才生成 `ConsumeQueue`。

```Java
  1: // ⬇️⬇️⬇️【DefaultMessageStore.java】
  2: public void doDispatch(DispatchRequest req) {
  3:     // 非事务消息 或 事务提交消息 建立 消息位置信息 到 ConsumeQueue
  4:     final int tranType = MessageSysFlag.getTransactionValue(req.getSysFlag());
  5:     switch (tranType) {
  6:         case MessageSysFlag.TRANSACTION_NOT_TYPE: // 非事务消息
  7:         case MessageSysFlag.TRANSACTION_COMMIT_TYPE: // 事务消息COMMIT
  8:             DefaultMessageStore.this.putMessagePositionInfo(req.getTopic(), req.getQueueId(), req.getCommitLogOffset(), req.getMsgSize(),
  9:                 req.getTagsCode(), req.getStoreTimestamp(), req.getConsumeQueueOffset());
 10:             break;
 11:         case MessageSysFlag.TRANSACTION_PREPARED_TYPE: // 事务消息PREPARED
 12:         case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE: // 事务消息ROLLBACK
 13:             break;
 14:     }
 15:     // 省略代码 =》 建立 索引信息 到 IndexFile
 16: }
```

# 3. 事务消息回查

* 【事务消息回查】功能曾经开源过，目前（V4.0.0）暂未开源。如下是该功能的开源情况：

| 版本 | 【事务消息回查】 | |
| --- | --- | --- |
| 官方V3.0.4 ~ V3.1.4 | 基于 文件系统 实现 | 已开源 |
| 官方V3.1.5 ~ V4.0.0 | 基于 数据库 实现 | 未完全开源 |

我们来看看两种情况下是怎么实现的。

## 3.1 Broker 发起【事务消息回查】

### 3.1.1 官方V3.1.4：基于文件系统

> 仓库地址：https://github.com/YunaiV/rocketmq-3.1.9/tree/release_3.1.4

相较于普通消息，【事务消息】多依赖如下三个组件：

* **TransactionStateService** ：事务状态服务，负责对【事务消息】进行管理，包括存储与更新事务消息状态、回查事务消息状态等等。
* **TranStateTable** ：【事务消息】状态存储。基于 `MappedFileQueue` 实现，默认存储路径为 `~/store/transaction/statetable`，每条【事务消息】状态存储结构如下：

| 第几位 | 字段 | 说明 | 数据类型 | 字节数 |
| :-- | :-- | :-- | :-- | :-- |
| 1 | offset | CommitLog 物理存储位置 | Long | 8 |
| 2 | size | 消息长度 | Int | 4 |
| 3 | timestamp | 消息存储时间，单位：秒 | Int | 4 |
| 4 | producerGroupHash | producerGroup 求 HashCode | Int | 4 |
| 5 | state | 事务状态 | Int | 4 |

* **TranRedoLog** ：`TranStateTable` 重放日志，每次**写操作** `TranStateTable` 记录重放日志。当 `Broker` 异常关闭时，使用 `TranRedoLog` 恢复 `TranStateTable`。基于 `ConsumeQueue` 实现，`Topic` 为 `TRANSACTION_REDOLOG_TOPIC_XXXX`，默认存储路径为 `~/store/transaction/redolog`。

-------

简单手绘逻辑图如下😈：

![Broker_V3.1.4_基于文件系统](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1011/Broker_V3.1.4_基于文件系统.jpeg)

#### 3.1.1.1 存储消息到 CommitLog

* 🦅存储【half消息】到 `CommitLog` 时，消息队列位置（`queueOffset`）使用 `TranStateTable` 最大物理位置（可写入物理位置）。这样，消息可以索引到自己对应的 `TranStateTable` 的位置和记录。

核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【DefaultAppendMessageCallback.java】
  2: class DefaultAppendMessageCallback implements AppendMessageCallback {
  3:     public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer,  final int maxBlank, final Object msg) {
  4:         // ...省略代码
  5: 
  6:         // 事务消息需要特殊处理 
  7:         final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
  8:         switch (tranType) {
  9:         case MessageSysFlag.TransactionPreparedType: // 消息队列位置（queueOffset）使用 TranStateTable 最大物理位置（可写入物理位置） 
 10:             queueOffset = CommitLog.this.defaultMessageStore.getTransactionStateService().getTranStateTableOffset().get();
 11:             break;
 12:         case MessageSysFlag.TransactionRollbackType:
 13:             queueOffset = msgInner.getQueueOffset();
 14:             break;
 15:         case MessageSysFlag.TransactionNotType:
 16:         case MessageSysFlag.TransactionCommitType:
 17:         default:
 18:             break;
 19:         }
 20: 
 21:         // ...省略代码
 22: 
 23:         switch (tranType) {
 24:         case MessageSysFlag.TransactionPreparedType:
 25:             // 更新 TranStateTable 最大物理位置（可写入物理位置） 
 26:             CommitLog.this.defaultMessageStore.getTransactionStateService().getTranStateTableOffset().incrementAndGet();
 27:             break;
 28:         case MessageSysFlag.TransactionRollbackType:
 29:             break;
 30:         case MessageSysFlag.TransactionNotType:
 31:         case MessageSysFlag.TransactionCommitType:
 32:             // 更新下一次的ConsumeQueue信息
 33:             CommitLog.this.topicQueueTable.put(key, ++queueOffset);
 34:             break;
 35:         default:
 36:             break;
 37:         }
 38: 
 39:         // 返回结果
 40:         return result;
 41:     }
 42: }
```

#### 3.1.1.2 写【事务消息】状态存储（TranStateTable）

* 🦅处理【Half消息】时，新增【事务消息】状态存储（`TranStateTable`）。
* 🦅处理【Commit / Rollback消息】时，更新 【事务消息】状态存储（`TranStateTable`） COMMIT / ROLLBACK。
* 🦅每次**写操作【**事务消息】状态存储（`TranStateTable`），记录重放日志（`TranRedoLog`）。

核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【DispatchMessageService.java】
  2: private void doDispatch() {
  3:     if (!this.requestsRead.isEmpty()) {
  4:         for (DispatchRequest req : this.requestsRead) {
  5: 
  6:             // ...省略代码
  7: 
  8:             // 2、写【事务消息】状态存储（TranStateTable）
  9:             if (req.getProducerGroup() != null) {
 10:                 switch (tranType) {
 11:                 case MessageSysFlag.TransactionNotType:
 12:                     break;
 13:                 case MessageSysFlag.TransactionPreparedType:
 14:                     // 新增 【事务消息】状态存储（TranStateTable）
 15:                     DefaultMessageStore.this.getTransactionStateService().appendPreparedTransaction(
 16:                         req.getCommitLogOffset(), req.getMsgSize(), (int) (req.getStoreTimestamp() / 1000), req.getProducerGroup().hashCode());
 17:                     break;
 18:                 case MessageSysFlag.TransactionCommitType:
 19:                 case MessageSysFlag.TransactionRollbackType:
 20:                     // 更新 【事务消息】状态存储（TranStateTable） COMMIT / ROLLBACK
 21:                     DefaultMessageStore.this.getTransactionStateService().updateTransactionState(
 22:                         req.getTranStateTableOffset(), req.getPreparedTransactionOffset(), req.getProducerGroup().hashCode(), tranType);
 23:                     break;
 24:                 }
 25:             }
 26:             // 3、记录 TranRedoLog
 27:             switch (tranType) {
 28:             case MessageSysFlag.TransactionNotType:
 29:                 break;
 30:             case MessageSysFlag.TransactionPreparedType:
 31:                 // 记录 TranRedoLog
 32:                 DefaultMessageStore.this.getTransactionStateService().getTranRedoLog().putMessagePostionInfoWrapper(
 33:                         req.getCommitLogOffset(), req.getMsgSize(), TransactionStateService.PreparedMessageTagsCode,
 34:                         req.getStoreTimestamp(), 0L);
 35:                 break;
 36:             case MessageSysFlag.TransactionCommitType:
 37:             case MessageSysFlag.TransactionRollbackType:
 38:                 // 记录 TranRedoLog
 39:                 DefaultMessageStore.this.getTransactionStateService().getTranRedoLog().putMessagePostionInfoWrapper(
 40:                         req.getCommitLogOffset(), req.getMsgSize(), req.getPreparedTransactionOffset(),
 41:                         req.getStoreTimestamp(), 0L);
 42:                 break;
 43:             }
 44:         }
 45: 
 46:         // ...省略代码
 47:     }
 48: }
 49: // ⬇️⬇️⬇️【TransactionStateService.java】
 50: /**
 51:  * 新增事务状态
 52:  *
 53:  * @param clOffset commitLog 物理位置
 54:  * @param size 消息长度
 55:  * @param timestamp 消息存储时间
 56:  * @param groupHashCode groupHashCode
 57:  * @return 是否成功
 58:  */
 59: public boolean appendPreparedTransaction(//
 60:         final long clOffset,//
 61:         final int size,//
 62:         final int timestamp,//
 63:         final int groupHashCode//
 64: ) {
 65:     MapedFile mapedFile = this.tranStateTable.getLastMapedFile();
 66:     if (null == mapedFile) {
 67:         log.error("appendPreparedTransaction: create mapedfile error.");
 68:         return false;
 69:     }
 70: 
 71:     // 首次创建，加入定时任务中
 72:     if (0 == mapedFile.getWrotePostion()) {
 73:         this.addTimerTask(mapedFile);
 74:     }
 75: 
 76:     this.byteBufferAppend.position(0);
 77:     this.byteBufferAppend.limit(TSStoreUnitSize);
 78: 
 79:     // Commit Log Offset
 80:     this.byteBufferAppend.putLong(clOffset);
 81:     // Message Size
 82:     this.byteBufferAppend.putInt(size);
 83:     // Timestamp
 84:     this.byteBufferAppend.putInt(timestamp);
 85:     // Producer Group Hashcode
 86:     this.byteBufferAppend.putInt(groupHashCode);
 87:     // Transaction State
 88:     this.byteBufferAppend.putInt(MessageSysFlag.TransactionPreparedType);
 89: 
 90:     return mapedFile.appendMessage(this.byteBufferAppend.array());
 91: }
 92: 
 93: /**
 94:  * 更新事务状态
 95:  *
 96:  * @param tsOffset tranStateTable 物理位置
 97:  * @param clOffset commitLog 物理位置
 98:  * @param groupHashCode groupHashCode
 99:  * @param state 事务状态
100:  * @return 是否成功
101:  */
102: public boolean updateTransactionState(
103:         final long tsOffset,
104:         final long clOffset,
105:         final int groupHashCode,
106:         final int state) {
107:     SelectMapedBufferResult selectMapedBufferResult = this.findTransactionBuffer(tsOffset);
108:     if (selectMapedBufferResult != null) {
109:         try {
110: 
111:             // ....省略代码：校验是否能够更新
112: 
113:             // 更新事务状态
114:             selectMapedBufferResult.getByteBuffer().putInt(TS_STATE_POS, state);
115:         }
116:         catch (Exception e) {
117:             log.error("updateTransactionState exception", e);
118:         }
119:         finally {
120:             selectMapedBufferResult.release();
121:         }
122:     }
123: 
124:     return false;
125: }
```

#### 3.1.1.3 【事务消息】回查

* 🦅`TranStateTable` 每个 `MappedFile` 都对应一个 `Timer`。`Timer` 固定周期（默认：60s）遍历 `MappedFile`，查找【half消息】，向 `Producer` 发起【事务消息】回查请求。【事务消息】回查结果的逻辑不在此处进行，在 [CommitLog dispatch](#3112-写事务消息状态存储transtatetable)时执行。

实现代码如下：

```Java
  1: // ⬇️⬇️⬇️【TransactionStateService.java】
  2: /**
  3:  * 初始化定时任务
  4:  */
  5: private void initTimerTask() {
  6:     //
  7:     final List<MapedFile> mapedFiles = this.tranStateTable.getMapedFiles();
  8:     for (MapedFile mf : mapedFiles) {
  9:         this.addTimerTask(mf);
 10:     }
 11: }
 12: 
 13: /**
 14:  * 每个文件初始化定时任务
 15:  * @param mf 文件
 16:  */
 17: private void addTimerTask(final MapedFile mf) {
 18:     this.timer.scheduleAtFixedRate(new TimerTask() {
 19:         private final MapedFile mapedFile = mf;
 20:         private final TransactionCheckExecuter transactionCheckExecuter = TransactionStateService.this.defaultMessageStore.getTransactionCheckExecuter();
 21:         private final long checkTransactionMessageAtleastInterval = TransactionStateService.this.defaultMessageStore.getMessageStoreConfig()
 22:                     .getCheckTransactionMessageAtleastInterval();
 23:         private final boolean slave = TransactionStateService.this.defaultMessageStore.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE;
 24: 
 25:         @Override
 26:         public void run() {
 27:             // Slave不需要回查事务状态
 28:             if (slave) {
 29:                 return;
 30:             }
 31:             // Check功能是否开启
 32:             if (!TransactionStateService.this.defaultMessageStore.getMessageStoreConfig()
 33:                 .isCheckTransactionMessageEnable()) {
 34:                 return;
 35:             }
 36: 
 37:             try {
 38:                 SelectMapedBufferResult selectMapedBufferResult = mapedFile.selectMapedBuffer(0);
 39:                 if (selectMapedBufferResult != null) {
 40:                     long preparedMessageCountInThisMapedFile = 0; // 回查的【half消息】数量
 41:                     int i = 0;
 42:                     try {
 43:                         // 循环每条【事务消息】状态，对【half消息】进行回查
 44:                         for (; i < selectMapedBufferResult.getSize(); i += TSStoreUnitSize) {
 45:                             selectMapedBufferResult.getByteBuffer().position(i);
 46: 
 47:                             // Commit Log Offset
 48:                             long clOffset = selectMapedBufferResult.getByteBuffer().getLong();
 49:                             // Message Size
 50:                             int msgSize = selectMapedBufferResult.getByteBuffer().getInt();
 51:                             // Timestamp
 52:                             int timestamp = selectMapedBufferResult.getByteBuffer().getInt();
 53:                             // Producer Group Hashcode
 54:                             int groupHashCode = selectMapedBufferResult.getByteBuffer().getInt();
 55:                             // Transaction State
 56:                             int tranType = selectMapedBufferResult.getByteBuffer().getInt();
 57: 
 58:                             // 已经提交或者回滚的消息跳过
 59:                             if (tranType != MessageSysFlag.TransactionPreparedType) {
 60:                                 continue;
 61:                             }
 62: 
 63:                             // 遇到时间不符合最小轮询间隔，终止
 64:                             long timestampLong = timestamp * 1000;
 65:                             long diff = System.currentTimeMillis() - timestampLong;
 66:                             if (diff < checkTransactionMessageAtleastInterval) {
 67:                                 break;
 68:                             }
 69: 
 70:                             preparedMessageCountInThisMapedFile++;
 71: 
 72:                             // 回查Producer
 73:                             try {
 74:                                 this.transactionCheckExecuter.gotoCheck(groupHashCode, getTranStateOffset(i), clOffset, msgSize);
 75:                             } catch (Exception e) {
 76:                                 tranlog.warn("gotoCheck Exception", e);
 77:                             }
 78:                         }
 79: 
 80:                         // 无回查的【half消息】数量，且遍历完，则终止定时任务
 81:                         if (0 == preparedMessageCountInThisMapedFile //
 82:                                 && i == mapedFile.getFileSize()) {
 83:                             tranlog.info("remove the transaction timer task, because no prepared message in this mapedfile[{}]", mapedFile.getFileName());
 84:                             this.cancel();
 85:                         }
 86:                     } finally {
 87:                         selectMapedBufferResult.release();
 88:                     }
 89: 
 90:                     tranlog.info("the transaction timer task execute over in this period, {} Prepared Message: {} Check Progress: {}/{}", mapedFile.getFileName(),//
 91:                             preparedMessageCountInThisMapedFile, i / TSStoreUnitSize, mapedFile.getFileSize() / TSStoreUnitSize);
 92:                 } else if (mapedFile.isFull()) {
 93:                     tranlog.info("the mapedfile[{}] maybe deleted, cancel check transaction timer task", mapedFile.getFileName());
 94:                     this.cancel();
 95:                     return;
 96:                 }
 97:             } catch (Exception e) {
 98:                 log.error("check transaction timer task Exception", e);
 99:             }
100:         }
101: 
102: 
103:         private long getTranStateOffset(final long currentIndex) {
104:             long offset = (this.mapedFile.getFileFromOffset() + currentIndex) / TransactionStateService.TSStoreUnitSize;
105:             return offset;
106:         }
107:     }, 1000 * 60, this.defaultMessageStore.getMessageStoreConfig().getCheckTransactionMessageTimerInterval());
108: }
109: 
110: // 【DefaultTransactionCheckExecuter.java】
111: @Override
112: public void gotoCheck(int producerGroupHashCode, long tranStateTableOffset, long commitLogOffset,
113:         int msgSize) {
114:     // 第一步、查询Producer
115:     final ClientChannelInfo clientChannelInfo = this.brokerController.getProducerManager().pickProducerChannelRandomly(producerGroupHashCode);
116:     if (null == clientChannelInfo) {
117:         log.warn("check a producer transaction state, but not find any channel of this group[{}]", producerGroupHashCode);
118:         return;
119:     }
120: 
121:     // 第二步、查询消息
122:     SelectMapedBufferResult selectMapedBufferResult = this.brokerController.getMessageStore().selectOneMessageByOffset(commitLogOffset, msgSize);
123:     if (null == selectMapedBufferResult) {
124:         log.warn("check a producer transaction state, but not find message by commitLogOffset: {}, msgSize: ", commitLogOffset, msgSize);
125:         return;
126:     }
127: 
128:     // 第三步、向Producer发起请求
129:     final CheckTransactionStateRequestHeader requestHeader = new CheckTransactionStateRequestHeader();
130:     requestHeader.setCommitLogOffset(commitLogOffset);
131:     requestHeader.setTranStateTableOffset(tranStateTableOffset);
132:     this.brokerController.getBroker2Client().checkProducerTransactionState(clientChannelInfo.getChannel(), requestHeader, selectMapedBufferResult);
133: }
```

#### 3.1.1.4 初始化【事务消息】状态存储（TranStateTable）

* 🦅根据最后 Broker 关闭是否正常，会有不同的初始化方式。

核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【TransactionStateService.java】
  2: /**
  3:  * 初始化 TranRedoLog
  4:  * @param lastExitOK 是否正常退出
  5:  */
  6: public void recoverStateTable(final boolean lastExitOK) {
  7:     if (lastExitOK) {
  8:         this.recoverStateTableNormal();
  9:     } else {
 10:         // 第一步，删除State Table
 11:         this.tranStateTable.destroy();
 12:         // 第二步，通过RedoLog全量恢复StateTable
 13:         this.recreateStateTable();
 14:     }
 15: }
 16: 
 17: /**
 18:  * 扫描 TranRedoLog 重建 StateTable
 19:  */
 20: private void recreateStateTable() {
 21:     this.tranStateTable = new MapedFileQueue(StorePathConfigHelper.getTranStateTableStorePath(defaultMessageStore
 22:                 .getMessageStoreConfig().getStorePathRootDir()), defaultMessageStore
 23:                 .getMessageStoreConfig().getTranStateTableMapedFileSize(), null);
 24: 
 25:     final TreeSet<Long> preparedItemSet = new TreeSet<Long>();
 26: 
 27:     // 第一步，从头扫描RedoLog
 28:     final long minOffset = this.tranRedoLog.getMinOffsetInQuque();
 29:     long processOffset = minOffset;
 30:     while (true) {
 31:         SelectMapedBufferResult bufferConsumeQueue = this.tranRedoLog.getIndexBuffer(processOffset);
 32:         if (bufferConsumeQueue != null) {
 33:             try {
 34:                 long i = 0;
 35:                 for (; i < bufferConsumeQueue.getSize(); i += ConsumeQueue.CQStoreUnitSize) {
 36:                     long offsetMsg = bufferConsumeQueue.getByteBuffer().getLong();
 37:                     int sizeMsg = bufferConsumeQueue.getByteBuffer().getInt();
 38:                     long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();
 39: 
 40:                     if (TransactionStateService.PreparedMessageTagsCode == tagsCode) { // Prepared
 41:                         preparedItemSet.add(offsetMsg);
 42:                     } else { // Commit/Rollback
 43:                         preparedItemSet.remove(tagsCode);
 44:                     }
 45:                 }
 46: 
 47:                 processOffset += i;
 48:             } finally { // 必须释放资源
 49:                 bufferConsumeQueue.release();
 50:             }
 51:         } else {
 52:             break;
 53:         }
 54:     }
 55:     log.info("scan transaction redolog over, End offset: {},  Prepared Transaction Count: {}", processOffset, preparedItemSet.size());
 56: 
 57:     // 第二步，重建StateTable
 58:     Iterator<Long> it = preparedItemSet.iterator();
 59:     while (it.hasNext()) {
 60:         Long offset = it.next();
 61:         MessageExt msgExt = this.defaultMessageStore.lookMessageByOffset(offset);
 62:         if (msgExt != null) {
 63:             this.appendPreparedTransaction(msgExt.getCommitLogOffset(), msgExt.getStoreSize(),
 64:                 (int) (msgExt.getStoreTimestamp() / 1000),
 65:                 msgExt.getProperty(MessageConst.PROPERTY_PRODUCER_GROUP).hashCode());
 66:             this.tranStateTableOffset.incrementAndGet();
 67:         }
 68:     }
 69: }
 70: 
 71: /**
 72:  * 加载（解析）TranStateTable 的 MappedFile
 73:  * 1. 清理多余 MappedFile，设置最后一个 MappedFile的写入位置(position
 74:  * 2. 设置 TanStateTable 最大物理位置（可写入位置）
 75:  */
 76: private void recoverStateTableNormal() {
 77:     final List<MapedFile> mapedFiles = this.tranStateTable.getMapedFiles();
 78:     if (!mapedFiles.isEmpty()) {
 79:         // 从倒数第三个文件开始恢复
 80:         int index = mapedFiles.size() - 3;
 81:         if (index < 0) {
 82:             index = 0;
 83:         }
 84: 
 85:         int mapedFileSizeLogics = this.tranStateTable.getMapedFileSize();
 86:         MapedFile mapedFile = mapedFiles.get(index);
 87:         ByteBuffer byteBuffer = mapedFile.sliceByteBuffer();
 88:         long processOffset = mapedFile.getFileFromOffset();
 89:         long mapedFileOffset = 0;
 90:         while (true) {
 91:             for (int i = 0; i < mapedFileSizeLogics; i += TSStoreUnitSize) {
 92: 
 93:                 final long clOffset_read = byteBuffer.getLong();
 94:                 final int size_read = byteBuffer.getInt();
 95:                 final int timestamp_read = byteBuffer.getInt();
 96:                 final int groupHashCode_read = byteBuffer.getInt();
 97:                 final int state_read = byteBuffer.getInt();
 98: 
 99:                 boolean stateOK = false;
100:                 switch (state_read) {
101:                 case MessageSysFlag.TransactionPreparedType:
102:                 case MessageSysFlag.TransactionCommitType:
103:                 case MessageSysFlag.TransactionRollbackType:
104:                     stateOK = true;
105:                     break;
106:                 default:
107:                     break;
108:                 }
109: 
110:                 // 说明当前存储单元有效
111:                 if (clOffset_read >= 0 && size_read > 0 && stateOK) {
112:                     mapedFileOffset = i + TSStoreUnitSize;
113:                 } else {
114:                     log.info("recover current transaction state table file over,  " + mapedFile.getFileName() + " "
115:                             + clOffset_read + " " + size_read + " " + timestamp_read);
116:                     break;
117:                 }
118:             }
119: 
120:             // 走到文件末尾，切换至下一个文件
121:             if (mapedFileOffset == mapedFileSizeLogics) {
122:                 index++;
123:                 if (index >= mapedFiles.size()) { // 循环while结束
124:                     log.info("recover last transaction state table file over, last maped file " + mapedFile.getFileName());
125:                     break;
126:                 } else { // 切换下一个文件
127:                     mapedFile = mapedFiles.get(index);
128:                     byteBuffer = mapedFile.sliceByteBuffer();
129:                     processOffset = mapedFile.getFileFromOffset();
130:                     mapedFileOffset = 0;
131:                     log.info("recover next transaction state table file, " + mapedFile.getFileName());
132:                 }
133:             } else {
134:                 log.info("recover current transaction state table queue over " + mapedFile.getFileName() + " " + (processOffset + mapedFileOffset));
135:                 break;
136:             }
137:         }
138: 
139:         // 清理多余 MappedFile，设置最后一个 MappedFile的写入位置(position
140:         processOffset += mapedFileOffset;
141:         this.tranStateTable.truncateDirtyFiles(processOffset);
142: 
143:         // 设置 TanStateTable 最大物理位置（可写入位置）
144:         this.tranStateTableOffset.set(this.tranStateTable.getMaxOffset() / TSStoreUnitSize);
145:         log.info("recover normal over, transaction state table max offset: {}", this.tranStateTableOffset.get());
146:     }
147: }
```

#### 3.1.1.5 补充

* 为什么 V3.1.5 开始，使用 数据库 实现【事务状态】的存储？如下是来自官方文档的说明，可能是一部分原因：

> RocketMQ 这种实现事务方式，没有通过 KV 存储做，而是通过 Offset 方式，存在一个显著缺陷，即通过 Offset 更改数据，会令系统的脏页过多，需要特别关注。

### 3.1.2 官方V4.0.0：基于数据库

> 仓库地址：https://github.com/apache/incubator-rocketmq

官方V4.0.0 暂时未**完全**开源【事务消息回查】功能，**So 我们需要进行一些猜想，可能不一定正确😈**。

😆我们来对比【官方V3.1.4：基于文件】的实现。

* TransactionRecord ：记录每条【事务消息】。类似 `TranStateTable`。

| TranStateTable | TransactionRecord |  |
| --- | --- | --- |
| offset | offset |  |
| producerGroupHash | producerGroup |  |
| size | 无 | 非必须字段：【事务消息】回查时，使用 offset 读取 CommitLog 获得。 |
| timestamp | 无 | 非必须字段：【事务消息】回查时，使用 offset 读取 CommitLog 获得。 |
| state | 无 | 非必须字段： 事务开始，增加记录；事务结束，删除记录。|

另外，数据库本身保证了数据存储的可靠性，无需 `TranRedoLog`。

-------

简单手绘逻辑图如下😈：

![Broker_V4.0.0_基于数据库](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1011/Broker_V4.0.0_基于数据库.jpeg)

## 3.2 Producer 接收【事务消息回查】

* 顺序图如下：

![Producer接收【事务消息回查】](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1011/Producer接收【事务消息回查】.png)

* 核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【DefaultMQProducerImpl.java】
  2: /**
  3:  * 检查【事务状态】状态
  4:  *
  5:  * @param addr broker地址
  6:  * @param msg 消息
  7:  * @param header 请求
  8:  */
  9: @Override
 10: public void checkTransactionState(final String addr, final MessageExt msg, final CheckTransactionStateRequestHeader header) {
 11:     Runnable request = new Runnable() {
 12:         private final String brokerAddr = addr;
 13:         private final MessageExt message = msg;
 14:         private final CheckTransactionStateRequestHeader checkRequestHeader = header;
 15:         private final String group = DefaultMQProducerImpl.this.defaultMQProducer.getProducerGroup();
 16: 
 17:         @Override
 18:         public void run() {
 19:             TransactionCheckListener transactionCheckListener = DefaultMQProducerImpl.this.checkListener();
 20:             if (transactionCheckListener != null) {
 21:                 // 获取事务执行状态
 22:                 LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
 23:                 Throwable exception = null;
 24:                 try {
 25:                     localTransactionState = transactionCheckListener.checkLocalTransactionState(message);
 26:                 } catch (Throwable e) {
 27:                     log.error("Broker call checkTransactionState, but checkLocalTransactionState exception", e);
 28:                     exception = e;
 29:                 }
 30: 
 31:                 // 处理事务结果，提交消息 COMMIT / ROLLBACK
 32:                 this.processTransactionState(//
 33:                     localTransactionState, //
 34:                     group, //
 35:                     exception);
 36:             } else {
 37:                 log.warn("checkTransactionState, pick transactionCheckListener by group[{}] failed", group);
 38:             }
 39:         }
 40: 
 41:         /**
 42:          * 处理事务结果，提交消息 COMMIT / ROLLBACK
 43:          *
 44:          * @param localTransactionState 【本地事务】状态
 45:          * @param producerGroup producerGroup
 46:          * @param exception 检查【本地事务】状态发生的异常
 47:          */
 48:         private void processTransactionState(//
 49:             final LocalTransactionState localTransactionState, //
 50:             final String producerGroup, //
 51:             final Throwable exception) {
 52:             final EndTransactionRequestHeader thisHeader = new EndTransactionRequestHeader();
 53:             thisHeader.setCommitLogOffset(checkRequestHeader.getCommitLogOffset());
 54:             thisHeader.setProducerGroup(producerGroup);
 55:             thisHeader.setTranStateTableOffset(checkRequestHeader.getTranStateTableOffset());
 56:             thisHeader.setFromTransactionCheck(true);
 57: 
 58:             // 设置消息编号
 59:             String uniqueKey = message.getProperties().get(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
 60:             if (uniqueKey == null) {
 61:                 uniqueKey = message.getMsgId();
 62:             }
 63:             thisHeader.setMsgId(uniqueKey);
 64: 
 65:             thisHeader.setTransactionId(checkRequestHeader.getTransactionId());
 66:             switch (localTransactionState) {
 67:                 case COMMIT_MESSAGE:
 68:                     thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
 69:                     break;
 70:                 case ROLLBACK_MESSAGE:
 71:                     thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
 72:                     log.warn("when broker check, client rollback this transaction, {}", thisHeader);
 73:                     break;
 74:                 case UNKNOW:
 75:                     thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
 76:                     log.warn("when broker check, client does not know this transaction state, {}", thisHeader);
 77:                     break;
 78:                 default:
 79:                     break;
 80:             }
 81: 
 82:             String remark = null;
 83:             if (exception != null) {
 84:                 remark = "checkLocalTransactionState Exception: " + RemotingHelper.exceptionSimpleDesc(exception);
 85:             }
 86: 
 87:             try {
 88:                 // 提交消息 COMMIT / ROLLBACK
 89:                 DefaultMQProducerImpl.this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, thisHeader, remark,
 90:                     3000);
 91:             } catch (Exception e) {
 92:                 log.error("endTransactionOneway exception", e);
 93:             }
 94:         }
 95:     };
 96: 
 97:     // 提交执行
 98:     this.checkExecutor.submit(request);
 99: }
100: 
101: // ⬇️⬇️⬇️【DefaultMQProducerImpl.java】
102: /**
103:  * 【事务消息回查】检查监听器
104:  */
105: public interface TransactionCheckListener {
106: 
107:     /**
108:      * 获取（检查）【本地事务】状态
109:      *
110:      * @param msg 消息
111:      * @return 事务状态
112:      */
113:     LocalTransactionState checkLocalTransactionState(final MessageExt msg);
114: 
115: }
```

