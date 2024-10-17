
---
title: "OpenIM源码分析: msgtransfer"
date: 2024-09-24T08:57:32+08:00
draft: false
categories:
 - open-im-server
tags:
 - golang
---
## tools/batcher

batcher的作用是scheduler从data 管道中每隔interval或者累积读取size个数据后，将数据发送到chArrays管道数组，
chArrays管道数组中的每个管道对应一个worker，worker在run之后，不断地从管道中读取msg，调用Do函数来处理。
最后可以通过Put方法向bacher发送数据

需要注意一下一些细节：

- scheduler从data管道收到数据后会通过Key函数，生成data地key，将数据添加到vals(map[key][]*T)中
- 上一步生成地vals相当于一批数据，发送到distributeMessage后，生成随机地triggerID，在处理前调用HooKFunc,
在处理后调用OnComplete,需要留意的是，如果syncWait参数被设置，只有在一批数据处理完之后才会调用OnComplete .在处理vals中每个(key, []*T)时，调用b.Sharding(key)确定这个key对应的一组数据，发送给
chArrays中的哪个通道。
- 关于停止batcher， 在调用Close后，b.cancel()会使b.globalCtx失效，导致Put会失败; b.data<-nil会导致scheduler循环直接返回，
defer中关闭b.chArrays中的每一个通道，导致每个worker返回.

## msgtransfer 

### 创建
```go
	msgTransfer := &MsgTransfer{
		historyCH:      historyCH,
		historyMongoCH: historyMongoCH,
	}
```

### 启动
```go
	go m.historyCH.historyConsumerGroup.RegisterHandleAndConsumer(m.ctx, m.historyCH)
	go m.historyMongoCH.historyConsumerGroup.RegisterHandleAndConsumer(m.ctx, m.historyMongoCH)
	err := m.historyCH.redisMessageBatches.Start()
	if err != nil {
		return err
	}
```
### OnlineHistoryRedisConsumerHandler
- 创建historyConsumerGroup, 用于消费ToRedisTopic
在ConsumeClaim中读取主题上的msg，Put进入redisMessageBatches
- 创建redisMessageBatches, 在do方法中处理上一步Put的msg。
主要是消息分类，然后分别调用handleMsg和handleNotification方法

#### 详细代码流程

首先初始化消费者群组，group_id 为redis， 用于持续读取主题toRedis中的消息，Put进入redisMessageBatches。
然后初始化redisMessageBatches,它的作用是把Put进入它的ConsumerMessage,根据CosumerMessage.Key的值进行分批，
相同Key值的消息都会进入同一批消息，然后由Sharding方法交给不同的worker去处理。如果Key的值每个对话唯一，
可以确定每一批消息都是同一个对话的内容。

在do方法中worker进行一个批次的消息处理。具体如下：
首先提取每个kafka消息Header中的内容放进context,组成新的消息数组
```go
	ctxMessages := och.parseConsumerMessages(ctx, val.Val())
```
然后对于每个消息，如果msg.message.ContentType 是 constant.HasReadReceipt， 设置conversation,userid的hasReadSeq。
```go
	och.doSetReadSeq(ctx, ctxMessages)
```
然后对这个批次的消息进行分类：
```go
	storageMsgList, notStorageMsgList, storageNotificationList, notStorageNotificationList :=
		och.categorizeMessageLists(ctxMessages)
```
根据是消息类型生成conversationID:
```go
	conversationIDMsg := msgprocessor.GetChatConversationIDByMsg(ctxMessages[0].message)
	conversationIDNotification := msgprocessor.GetNotificationConversationIDByMsg(ctxMessages[0].message)
```

最后分别处理msg和notification
```go
	och.handleMsg(ctx, val.Key(), conversationIDMsg, storageMsgList, notStorageMsgList)
	och.handleNotification(ctx, val.Key(), conversationIDNotification, storageNotificationList, notStorageNotificationList)
```
+ handlerMsg 
  ```go
  	och.toPushTopic(ctx, key, conversationID, notStorageList)
  ```
  对于storageMessageList,首先插入将消息列表插入redis中：
  ```go
 lastSeq, isNewConversation, err := och.msgTransferDatabase.BatchInsertChat2Cache(ctx, conversationID, storageMessageList) 
  ```
  在这个函数中，会给这个会话分配MaxSeq，然后在之前的maxseq基础上给每个消息分配seq，之后把这些消息写入redis，
  最后更新用户的会话HasReadReqs。
  
  如果上一步返回的isNewConversation为true，根据消息的会话类型，调用conversationRpc的方法创建会话。
  
  最后写入ToMongoMQ
  ```go
  err = och.msgTransferDatabase.MsgToMongoMQ(ctx, key, conversationID, storageMessageList, lastSeq)
  
  ```
+ handleNotification
逻辑和上面的类似， 主要不用处理isNewConversation的逻辑



### OnlineHistoryMongoConsumerHandler

- 创建historyConsumerGroup, 用于消费ToMongoTopic
在ConsumeClaim中读取主题上的msg，调用handleChatWs2Mongo

#### 详细代码流程
首先初始化消费者群组，group_id 为mongo， 用于持续读取主题toMongo中的消息。
然后在handleChatWs2Mongo中处理消息，
最后调用
```go
err = mc.msgTransferDatabase.BatchInsertChat2DB(ctx, msgFromMQ.ConversationID, msgFromMQ.MsgData, msgFromMQ.LastSeq)
```
插入一批消息进入mongo
