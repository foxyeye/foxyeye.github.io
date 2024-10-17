
---
title: "OpenIM源码分析: push"
date: 2024-09-25T16:23:51+08:00
draft: false
categories:
 - open-im-server
tags:
 - golang
---

## golang-lru
internal 实现了一个双向链表LruList, 用于存储Entry[K, V], 作为实现simplelru和expirable的底层数据结构。
simplelru 实现了一个简单的LRU，size控制数量，onEvict当删除数据的时候调用, items结构体成员中存储map[K]*Entry[K,V],
方便查询。
expirable 实现了过期逻辑，Add(K, V)的时候Entry的ExpireAt设置为now+ttl,并加入到buckets中， NewLRU的时候如果设置的ttl>0, 会启动一个goroutine，每隔一段时间清理过期的bucket。buckets固定有100个。

## pkg/localcache
### lru_lazy.go
底层使用simplelru, V变成结构体
```go
type layLruItem[V any] struct {
	lock    sync.Mutex
	expires int64
	err     error
	value   V
}

```
expires用于设置value的过期时间，在查询K的时候，如果expires > time.Now(), 则说明没有过期，直接返回value和err。
过期时间分两种，成功的过期时间，失败的过期时间，对应fetch()函数返回成功还是失败，在K不存在或K已经过期的条件下，
都会调用fetch()函数获取最新值

### lru_expiration.go
底层使用expirable.LRU, 过期时间使用successTTL, failedTTL无用参数，功能和底层LRU没有大的区别。


### lru_slot.go
实质上就是创建slotNum个LRU，每次查询和设置都是调用hash(K)得到index, 然后调用slots[index]中的对应方法。

### link/link.go 

New(n)创建slot，n代表有linkKey的数量

```go
type slot struct {
	n     uint64
	slots []*linkKey
}

type linkKey struct {
	lock sync.Mutex
	data map[string]map[string]struct{}
}

```
调用linkKey的link方法关联key和它的子key们。
注意，调用slot的Link方法会key和它的子key们，也会反向关联子key到父key，而且关联根据index(key)的值，关联
可能分布在不同的slot中
```go
func (x *slot) Link(key string, link ...string) {
	if len(link) == 0 {
		return
	}
	mk := key
	lks := make([]string, len(link))
	for i, k := range link {
		lks[i] = k
	}
	x.slots[x.index(mk)].link(mk, lks...)
	for _, lk := range lks {
		x.slots[x.index(lk)].link(lk, mk)
	}
}

```

调用Del(key)删除一个key的时候，不仅删除key，而且它的子key也会被删除

### localcache/cache.go 
这个文件是以上介绍所有功能的集成者，
Get调用GetLink，区别在GetLink会关联key，和 links， 
Del和DelLocal, 区别是Del会调用opt.delFn, Del不仅删除参数keys，而且也会删除他们关联的子key在local中的值


## push代码逻辑分析

### push_handler.go

- 创建NewConsumerHandler

consumerHandler包含 onlineCache, 用于缓存map[userid][]platformids,如果配置文件中FullUserCache被设置，那么
采用localcache存储,直接订阅doSubscribe中订阅redis的online_change主题，在goroutine中不断更新缓存。

如果采用sync.Map作为缓存，首先调用initUsersOnlineStatus，在其中调用userRpcClient.GetAllOnlineUsers初始化缓存，
然后再调用doSbuscribe。

consumerHandler包含pushConsumerGroup, 处理配置文件中的主题ToPushTopic，即"toPush"。该主题的生产者为msgtransfer。
在ConsumerClaim中不断循环读取msg，调用handleMsg2PsChat。

handleMsg2PsChat中根据msg.SessionType的类型调用Push2Group或者Push2User。他们都会调用GetConnsAndOnlinePush方法，
在onlinepusher.go文件中有具体的逻辑，简单地说就是获取所有msggateway的rpc方法推送消息。然后就是处理offline的逻辑，
group和user消息类型的处理逻辑不一样，group最终会调用asyncOfflinePush，异步的往kafka的toOfflinePushTopic主题，即
"toOfflinePush"写数据。而user会调用offlinePushMsg方法，最终调用offlinePusher.Push方法推送数据。

offlinePusher的逻辑在offlinepush目录中，其中有几个实现，默认采用个推,主要采用http post的方式发送数据

- 创建NewOfflinePushConsumerHandler

主要的逻辑是订阅配置文件中的ToOfflinePushTopic， 即"toOfflinePush", 然后不断读取消息，最后调用offlinePusher.Push方法推送数据,默认使用个推的方式。





## 最后流程总结

### ConsumerHandler

创建kafka消费群组，消费群组group_id 为 push, 消费的主题为push。
开始消费主题之前需要初始化onlineCache, onlineCache根据配置文件中fullUserCache参数的值，
决定采用sync.Map还是localcache中的lru.NewSlotLRU存储user_id到platform_ids中的映射。
如果fullUserCache为true，那么需要调用userRpc中的GetAllOnlineUsers方法获取所用用户的在线数据。
最后在协程中调用doSubscribe订阅online_change主题，解析message.Payload，设置对应userID的platformIDS。
online_change主题的生产者在userRpc中的SetUserStatus和SetUserOnlineStatus中设置用户状态数据。

消费者群组中的ConsumeClaim方法中调用handleMs2PsChat，在该方法中首先反序列化得到
pbpush.PushMsgReq,其中包含ConversationID， msgData, UserIDs几个字段。
根据MsgData.SessionType的类型，决定处理逻辑。
如果是ReadGroupChatType, 调用c.Push2Group(ctx, msgFromMQ.MsgData.GroupID, msgFromMQ.MsgData),
否则调用c.Push2User(ctx, pushUserIDList, msgFromMQ.MsgData)

#### Push2Group
首先调用c.groupMessagesHandler(ctx, groupID, &pushToUserIDs, msg)，
在其中从groupRpc中调用GetGroupMemberIDs获取pushToUserIDs，根据msg.ContentType的类型作不同的处理：
- MemberQuitNotification

  DeleteMemberAndSetConversationSeq方法中调用msgRpc的GetConversationMaxSeq和conversationRpc的SetConversationMaxSeq
  *pushToUserIDs = append(*pushToUserIDs, tips.QuitUser.UserID)

- MemberKickedNotification

  同上也调用DeleteMemberAndSetConversationSeq
  *pushToUserIDs = append(*pushToUserIDs, kickedUsers...)

- GroupDismissedNotification

  c.groupRpcClient.DismissGroup(ctx, groupID);

然后调用wsResults, err := c.GetConnsAndOnlinePush(ctx, msg, pushToUserIDs)
在其中从onlineCache中获取onlineUserIDs, offlineUserIDs,
然后调用result, err = c.onlinePusher.GetConnsAndOnlinePush(ctx, msg, onlineUserIDs)
offlineUserIDS也放到result结构中:
```go
for _, userID := range offlineUserIDs {
		result = append(result, &msggateway.SingleMsgToUserResults{
			UserID: userID,
		})
	}

```
然后处理需要线下推送的逻辑，过滤需要调用conversationRpc的方法过滤出需要offlinePush的用户ID，
```go
needOfflinePushUserIDs, err := c.conversationRpcClient.GetConversationOfflinePushUserIDs(
		ctx, conversationutil.GenGroupConversationID(groupID), offlinePushUserIDs)
```

最后调用c.asyncOfflinePush(ctx, needOfflinePushUserIDs, msg),
在其中发送到主题toOfflinePush

```go
if err := c.pushDatabase.MsgToOfflinePushMQ(ctx, conversationutil.GenConversationUniqueKeyForSingle(msg.SendID, msg.RecvID), needOfflinePushUserIDs, msg); err != nil {
	log.ZError(ctx, "Msg To OfflinePush MQ error", err, "needOfflinePushUserIDs",
		needOfflinePushUserIDs, "msg", msg)
	prommetrics.SingleChatMsgProcessFailedCounter.Inc()
	return
}
```

#### PushUser

和PushGroup一样先调用
wsResults, err := c.GetConnsAndOnlinePush(ctx, msg, userIDs)
如果需要线下推送则调用:
err = c.offlinePushMsg(ctx, msg, offlinePushUserID), 在其中再调用
err = c.offlinePusher.Push(ctx, offlinePushUserIDs, title, content, opts), 和group不同，
没有先发送到kafka


### OfflinePushConsumer

首先初始化消费者群组， group_id 为offlinePush, 主题为toOfflinePush
然后调用o.handleMsg2OfflinePush(ctx, msg.Value),
最终调用err = o.offlinePusher.Push(ctx, offlinePushUserIDs, title, content, opts)
所以这个Consumer只是用于消费PushGroup中的线下消息，最终通过个推等方式推送到用户终端。


OnlinePusher会遍历etcd中所有的msggateway节点，调用每个节点的SuperGroupOnlineBatchPushOneMsg
