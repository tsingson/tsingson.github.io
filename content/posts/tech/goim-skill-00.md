---
title: "goim 架构与定制"
date: 2019-04-21T22:02:57+08:00
hidden: false
draft: false
tags: [golang, go, goim]
keywords: [tsingson]
description: "goim 架构与定制"
slug: "goim-go-01"
---



### 0. 关于 goim 及文章撰写动机

goim 官网 [http://goim.io](http://goim.io)

goim 源码 [https://github.com/Terry-Mao/goim](https://github.com/Terry-Mao/goim)

___


goim 是 非常成功的 IM (Instance Message) 即时消息平台, 依赖项为 kafka ( 消息队列) + zookeeper ( 扩展/均衡 ) + bilibili/discovery( 在 [netflix/eureka](https://github.com/Netflix/eureka)上扩展的服务注册与发现, [golang](https://golang.org) 实现) 


作为一个曾经的架构师(2005~2014, Utstarcom IPTV/OTT 事业部) 与当前自由职业者(你懂的~~~~), 时常在 Golang 圈转转,  有朋友聊到IM 并提到goim, 我作了一些学习与研究 

中国 B站( BiliBili ) 的技术领军 [毛剑](https://github.com/Terry-Mao/) 是我神交以久的技术专家,   [goim](https://github.com/Terry-Mao/goim)  是一个非常成功的架构示例, 其模块拆分, 接口设计, 技术选型 ,部署方式 以及持续改进演变, 都是一个互联网商用项目典范.  


同时,  另一位技术专家 [Xin.zh](https://github.com/alexstocks) 的文章 [一套高可用实时消息系统实现](https://alexstocks.github.io/html/pubsub.html)  给我很大启发.  

在电信/广电的几年经历, 这一次,  闲来无事, 算是满怀着在巨人肩头的感谢与敬意, 尝试写一些代码来加深学习.  



**感谢两位技术专家, 感谢开源社区**




个人在 Utstarcom 以业务平台架构师/解决方案工程师/ IPTV播控产品线 release manager 角色折腾过比较长一些时间,  除了技术方案的原型代码撰写与现场应急帮忙修bug 以外, 甚少参撰写商用项目中的代码,  这次写写代码也是有趣的练习  :P

欢迎指点/交流....



### 1. goim 原始构设计
  原图在这里
![original architecture](https://user-gold-cdn.xitu.io/2019/4/21/16a3ce2bb0f8cb43?w=761&h=506&f=png&s=25978)

  我重绘了这张图, 把各网元, 及外部网元关系标示清晰一些

  **说明**: 下图右侧 http client 是 goim push message 接口, 我标注了 backend 只是个人习惯, 事实上这只是个即时消息发送接口, 无所谓前后台


 ![original architecture](https://user-gold-cdn.xitu.io/2019/4/21/16a3ce9aca706cb9?w=1342&h=790&f=png&s=84094)

 #### 注意要点:
>
>1. comet / job / logic 支持多实例部署, 这是 goim 分布式架构设计的精粹. 同时, push message 消息发布接口从 comet 拆分也有一定的考量, 毕竟多数IM 尤其是 bilibili 的业务场景上来说, 发送量少, 而阅读量多, 想想弹幕的业务场景就明白了.
>
>2. goim 采用 [bilibili/discovery](https://github.com/bilibili/discovery) 实现注册/服务发现, 从而实现分布式路由与动态调度, 相关细节参看 [bilibili/discovery](https://github.com/bilibili/discovery) 文档, 以及 [Netflix/eureka](https://github.com/Netflix/eureka) 原始设计文档
>
>3. 配置 discovery 时, 注意 region / zone / env 的相互匹配对应关系
>
>4. 测试部署请注意 redis-server 尽量只要部署一个实例或一个集群(相当于单实例), kafka / zookeeper 相对简单, 部署多少都行, 配置对接上就行



### 2. 架构细节(内部逻辑组件与接口关系)

goim 源码不多, 阅读简单也算是 golang 语言的特点, 在 goim 尤其如此. ( 推荐用 goland 阅读代码)
下图中 goim 各网元的内部逻辑组件(逻辑单元), 以及各逻辑接口的相互关系, 可以对照源码自行阅读, 扩展

> 请注意各网元的连接线, 箭头标示了数据/信令的流向

 ![architecture degail ( original )](https://user-gold-cdn.xitu.io/2019/4/21/16a3ce9f075d93dd?w=2141&h=1096&f=png&s=159332)


### 3. 如何定制扩展

在学习过程中, 网上问到比较多的定制问题有几个, 分别如下

1. 离线消息如何存储
2. 用户如何认证, 或如何与自有业务系统对接
3. kafka 建议可更换, 比如 nats (我作了这个尝试)
4. bilibili/discovery 分离


下面画出 goim 定制扩展, 或优化的一个可行方式

![architecture degail ( original )](https://user-gold-cdn.xitu.io/2019/4/21/16a3d10a3902c923?w=1328&h=790&f=png&s=92400)

1. 在 comet 上定制扩展,  client 端增加消息发送, 可双向流式发送/接收即时消息
2. 在 logic 上定制, 增加用户管理接口, 会话管理接口, room 管理接口, 以及
3. 在 logic 上增加即时消息存储或处理接口, 比如离线消息存储, 用户上线后获取离线消息(后台触发发送)
4. 当然了, logic 上原有的 http client 发送接口保留




#### 0. 请注意
>
> 下面的源码标记出处在 [https://github.com/Terry-Mao/goim](https://github.com/Terry-Mao/goim) 
>
> 与我的 repo [https://github.com/tsingson/goim](https://github.com/tsingson/goim) 并不相同!!!
>
> 我 fork 的代码库中, 消息队列抽象成为golang 的 interface , 并且 discovery 正在抽离处理中


#### 1. 即时消息的存储钩子
源码在文件 /internal/logic/dao/kafka.go 中



```
// PushMsg push a message to databus.
func (d *Dao) PushMsg(c context.Context, op int32, server string, keys []string, msg []byte) (err error) {
	pushMsg := &pb.PushMsg{
		Type:      pb.PushMsg_PUSH,
		Operation: op,
		Server:    server,
		Keys:      keys,
		Msg:       msg,
	}

        //
        // 即时消息存储扩展 HOOKS:
        // 在这里增加即时消息存储扩展
        // 如果需要只存储离线消息, 可以先检查当前用户是否在线, 依据用户在线情况处理存储 
        //

	b, err := proto.Marshal(pushMsg)
	if err != nil {
		return
	}
	m := &sarama.ProducerMessage{
		Key:   sarama.StringEncoder(keys[0]),
		Topic: d.c.Kafka.Topic,
		Value: sarama.ByteEncoder(b),
	}
	if _, _, err = d.kafkaPub.SendMessage(m); err != nil {
		log.Errorf("PushMsg.send(push pushMsg:%v) error(%v)", pushMsg, err)
	}
	return
}
```

#### 2. 用户管理与会话管理
源码在 /internal/logic/conn.go

```

// Connect connected a conn.
func (l *Logic) Connect(c context.Context, server, cookie string, token []byte) (mid int64, key, roomID string, accepts []int32, hb int64, err error) {
	var params struct {
		Mid      int64   `json:"mid"`
		Key      string  `json:"key"`
		RoomID   string  `json:"room_id"`
		Platform string  `json:"platform"`
		Accepts  []int32 `json:"accepts"`
	}
	if err = json.Unmarshal(token, &params); err != nil {
		log.Errorf("json.Unmarshal(%s) error(%v)", token, err)
		return
	}
	mid = params.Mid
	roomID = params.RoomID
	accepts = params.Accepts
	hb = int64(l.c.Node.Heartbeat) * int64(l.c.Node.HeartbeatMax)
	// 
        //  用户管理 HOOKS
        //  这里增加用户管理逻辑代码, 比如:
        //  1. 调用用户管理模块( 比如 UMS)  检查 mid ( 会员ID / 用户 ID ) 是否存在
        //  2. 检查用户与 room 的权限关系
        //
        // 补充: 一般来说, goim 就作为一个即时消息服务, 用户注册/用户认证等业务应该由 goim 以外的网元或子系统完成
        // 这里的 HOOKS 只要提供一个与用户管理子系统/会话管理子系统的相应接口调用就可以了
        //


        // 会话管理 HOOKS
        // key 是会话ID ( session ID) , 在这里增加会话管理逻辑代码, 比如:
        // 1. 检查会话 ID 是否合法
        // 2. 如果不合法, 为授权用户创建会活ID
        //  
       // 下面这个 if 代码段, 是一个简化掉的例子: 
	if key = params.Key; key == "" {

		keyUuid, _ := uuid.NewV4()
		key = keyUuid.String()
	}

	// 这里是保存用户会话
	if err = l.dao.AddMapping(c, mid, key, server); err != nil {
		log.Errorf("l.dao.AddMapping(%d,%s,%s) error(%v)", mid, key, server, err)
		return
	}
	//
	log.Infof("conn connected key:%s server:%s mid:%d token:%s", key, server, mid, token)
	return
}
```

### 4. 我的定制扩展

由于学习目的, 及深入阅读源码的需求, 简化了 kafka / zk 的复杂部署参数配置与 jvm 依赖, 我 fork 了 goim 并修改为 [nats](https://github.com/nats-io/gnatsd) + ~~[liftbridge](https://github.com/liftbridge-io/liftbridge)~~, 由   [nats](https://github.com/nats-io/gnatsd) 实现 简化掉 kafka 队列功能 + zookeeper , ~~由  [liftbridge](https://github.com/liftbridge-io/liftbridge)  实现 nats 消息的持久化~~

源码见这里 [https://github.com/tsingson/goim](https://github.com/tsingson/goim) 上作一些扩展学习

---

### 关于我
网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  




 [tsingson](https://github.com/tsingson) 写于中国深圳, 2019/04/21