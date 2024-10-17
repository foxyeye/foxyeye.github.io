
---
title: "OpenIM源码分析: msggateway"
date: 2024-08-30T11:32:47+08:00
draft: false
categories:
 - open-im-server
tags:
 - golang
---

# 概要

msggateway由hubServer和longServer组成，hubServer为RPC Server， longServer为websocket Server。

# 启动过程

## hubServer启动过程

### 调用通用方法 startrpc.Start

1. 使用传入的rpc port和 listenIP 创建 net.listener

2. 创建discovery.SvcDiscoveryRegistry

3. SvcDiscoveryRegistry中添加  mw.GrpcClient() 返回的grpc.DialOption

   + 注册中间件RpcClientInterceptor, 该中间件会在调用实际方法之前，检查context之中有无operationID, 没有返回错误，
   + 检查context中是否存在OpUserID, opUserPlatform, ConnID, 如果存在, 放入metadata中，并生成
包含此metadata的新的context
   + invoke实际方法之后，检查返回的err，生成统一的CodeError

4. 判断传入的registerIP是否为空，如果为空，根据网口，生成新的registerIP

5. 判断是否启用Prometheus，如果启动，执行以下步骤1和2, 如果没有启用，只执行步骤1:

    1) 在ServerOptions中添加mw.GrpcServer()

    - 注册RpcServerInterceptor, 该中间件从context中获取metadata，检查是否包含OperationID，如果没有返回错误,

    - 从metadata中获取constant.RpcCustomHeader字符串数组，遍历其中的每一个key，根据这个key再从metadata中获取对应的value
        把key，value添加到context中

    - 从metadata中获取OperationID,OpUserPlatform和ConnID放到context中

    2) 在ServerOptions中添加prommetricsUnaryInterceptor(rpcRegisterName),该方法会在调用直接的rpc方法后, 增加rpcCounter这个指标，
        主要用于记录rpc调用的服务名、方法、返回code的次数

6. 将当前rpc服务注册到SvcDiscovoryRegistry中

7. 如果启用Prometheus

      根据rpcRegisterName配置指标(Metric), msggateway的指标为Gauge类型的online_user_num
        调用RpcInit方法，该方法又添加baseCollecotr和rpcCounter指标，最后调用Init方法启动httpServer暴漏/metrics路径

## LongConnServer启动过程

  1. ws 服务器启动之前，单独启动一个goroutine运行 ChangeOnlineStatus(), 该函数每过一段事件从ws.clients获取用户登录信息，
     或者当ws.clients.UserState()返回的channel有数据时，调用ws.userClient.Client.SetUserOnlineStatus，更新用户在线状态

  2. 启动一个goroutine 处理client的注册和删除，以及处理当新的client创建的时候，由于同一个platform只允许登录一个client的时候，
     会删除其他的client的情况
  3. 注册ws.wsHandler到"/"路径下，启动httpServer
  
  4. wsHandler的处理逻辑

      + 创建UserConnContext，记录respWriter， Request, Path, Method, RemoteAddr, 生成ConnID
      + 用户达到最大连接数，返回错误
      + UserConnContext.ParseEssentialArgs()校验token， sendID, platformID等参数不为空
      + ws.authClient.ParseToken 校验token
      + 校验上一步返回的userID和platformID和UserConnContext中的是否相等
      + newGWebSocket生成GWebSocket，用于封装websocket.Conn, 根据Query参数isMsgResp，决定是否返回ApiResponse成功信息，并退出
      + ws.clientPool中获取Client并reset，将此Client 发送到ws.registerChan，在协程中调用，client.readMessage()

  5. client.readMessage处理逻辑

      + 处理过程中如果发生错误，调用Client.close()
      在close中，会关闭底层wsconn, 调用wsServer的UnRegister(client)
      + 设置PingHandler, PongHandler, 如果c.platformID是WebPlatfromID那么在协程里持续writePingMsg
      + 循环调用c.conn.ReadMessage()，根据messageType处理信息，
      文本信息不支持，PingMessage调用c.writePingMsg，CloseMessage返回，MessageBinary调用c.hanleMessage

  6. client.handleMessage处理逻辑
     + 如果c.IsCompress， 需要先解压缩
     + 调用c.longConnServer.Decode(message, binaryReq)获取解码后的请求，请求必须要是Req结构体
     + 调用c.longConnServer.Validate(binaryReq)，当前没有校验逻辑
     + 校验binaryReq.SendID和c.UserID是否相等
     + 生成context包含binaryReq中的OperationID, SendID, c.PlatfromnID, c.ctx.GetConnID 
     + 根据binary.ReqIdentifier的类型，调用c.longConnServer的方法
       - GetSeq
       - SendMessage
       - SendSignalMessage
       - PullMessageBySeqList
       - UserLogout
       - setAppBackgroundStatus
       - SubUserOnlineStatus


##  启动过程中会设置WsServer的online

```golang
	hubServer := NewServer(rpcPort, longServer, conf, func(srv *Server) error {
		longServer.online = rpccache.NewOnlineCache(srv.userRcp, nil, rdb, longServer.subscriberUserOnlineStatusChanges)
		return nil
	})

```
NewServer中会在协程中循环从redis中的cachekey,onlineChannel中获取p1:p1:user1类似的数据，解析之后设置在lru.NewSlotLRU中，并且调用
longServer.subscriberUserOnlineStatusChanges, 此方法会调用ws.pushUserIDOnlineStatus, 此方法会根据userID，获取ws.subscription中存储的
订阅此userID的clients, 然后给每个client调用client.PushUserOnlineStatus
