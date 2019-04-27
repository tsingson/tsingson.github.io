---
title: "从goim定制, 浅谈 golang 的 interface 解耦合与gRPC"
date: 2019-04-23T22:02:57+08:00
hidden: false
draft: false
tags: [golang, go, goim]
keywords: [tsingson]
description: "从goim定制, 浅谈 golang 的 interface 解耦合与gRPC"
slug: "goim-go-02"
---

# 从goim定制, 浅谈 golang 的 interface 解耦合与gRPC

## 0. 背景及动机
继上一篇文章 [goim 架构与定制](https://juejin.im/post/5cbb9e68e51d456e51614aab) , 再谈 [goim](https://github.com/Terry-Mao/goim) 的定制扩展, 这一次谈一弹 goim 从 kafka 转到 nats 

github 上的 issue 在这里[https://github.com/Terry-Mao/goim/issues/262](https://github.com/Terry-Mao/goim/issues/262)


简要说明一下 golang 的 interface:
在 [吴德宝AllenWu](https://juejin.im/post/5a6873fd518825734501b3c5) 文章[Golang interface接口深入理解](https://juejin.im/post/5a6873fd518825734501b3c5) 中这样写到:


> 为什么要用接口呢？在Gopher China 上的分享中，有大神给出了下面的理由：
>
> > writing generic algorithm （类似泛型编程）
> >
> > hiding implementation detail （隐藏具体实现）
> >
> > providing interception points (提供拦截点-----> 也可称叫提供 HOOKS , 一个插入其他业务逻辑的钩子)

----

换个方式说, interface 就是 de-couple 解耦合在 golang 中的实施, 这是现代编程中比较重要的"分层, 解耦合" 架构设计方法
----


 在QQ群"golang中国" 中, 有关于 de-couple 解耦合的话题中, 闪侠这样说到:


![](https://user-gold-cdn.xitu.io/2019/4/22/16a433ad4d48cb54?w=762&h=408&f=png&s=50814)

![](https://user-gold-cdn.xitu.io/2019/4/22/16a433a9b899ecc4?w=956&h=448&f=png&s=61009)

这里, 就来看看 interface 如何实现 goim 从 [kafka](https://kafka.apache.org) 转到 [NATS](https://nats.io) 


## 1. goim 中的 kafka 

看图, 不说话, 哈哈


![](https://user-gold-cdn.xitu.io/2019/4/22/16a433e3b20e5a9f?w=1320&h=768&f=png&s=86616)

上图中,
1. 在 logic 这个网元中, 有 logic 向 kafka 的消息发布
2. 在 job 网元中, job 从 kafka 订阅消息, 再赂 comet 网元分发


那我们的目标很简单了, 换了!!! ----------> **等等**.......能保留原有 kafka 实现不? 在必要时, 可以使用开关项, 切换 nats 或 kafka ??

**当然......可以!**

----

 

## 2. Don't talk, show me the code!!
下面就比较简单, 看码

### 2.1 发布接口第一步, 阅读原代码

先看源代码( **注意下面代码中的注释**)


> 代码在 [https://github.com/Terry-Mao/goim/blob/master/internal/logic/push.go](https://github.com/Terry-Mao/goim/blob/master/internal/logic/push.go) 大约第33行

```
// PushMids push a message by mid.
func (l *Logic) PushMids(c context.Context, op int32, mids []int64, msg []byte) (err error) {
	keyServers, _, err := l.dao.KeysByMids(c, mids)
	if err != nil {
		return
	}
	keys := make(map[string][]string)
	for key, server := range keyServers {
		if key == "" || server == "" {
			log.Warningf("push key:%s server:%s is empty", key, server)
			continue
		}
		keys[server] = append(keys[server], key)
	}
	for server, keys := range keys {
	    // 
	    //  主要向 kafka 发送消息, 是下面这一行
	    //  l.dao.PushMsg(c, op, server, keys, msg)
	    //  方法名是 PushMsg
	    //
		if err = l.dao.PushMsg(c, op, server, keys, msg); err != nil {
			return
		}
	}
	return
}
```



再看一下 dao 是什么:

> 代码在 [https://github.com/Terry-Mao/goim/blob/master/internal/logic/logic.go](https://github.com/Terry-Mao/goim/blob/master/internal/logic/logic.go) 大约第20行

```
// Logic struct
type Logic struct {
	c   *conf.Config
	dis *naming.Discovery
	//
	//
	// 下面这个 dao.Dao 提供了 PushMsg 方法
	// 带个星, 这是个引用
	//
	//
	dao *dao.Dao
	// online
	totalIPs   int64
	totalConns int64
	roomCount  map[string]int32
	// load balancer
	nodes        []*naming.Instance
	loadBalancer *LoadBalancer
	regions      map[string]string // province -> region
}
```



最后, **重点来了**, 查到 dao 源头实现

>> 下面是我们需要扩展的地方, 在 [https://github.com/Terry-Mao/goim/blob/master/internal/logic/dao/](https://github.com/Terry-Mao/goim/blob/master/internal/logic/dao/)中
>> dao, 这名称很 java (DAO-------> Data Access Objects 数据存取对象), 这里也说明了 bilibili 们在代码纺织上, 挺规范


> 代码在 [https://github.com/Terry-Mao/goim/blob/master/internal/logic/dao/dao.go](https://github.com/Terry-Mao/goim/blob/master/internal/logic/dao/dao.go) 大约第10行开始

```
// Dao dao.
type Dao struct {
	c           *conf.Config
	//
	// ******************************************************************
	// 下面这个 kafkaPub 很清楚, 是 kafka 的同步发布者 kafka.SyncProducer
	// 
	//  这个是我们要换成 interface 的地方
	//
	// ******************************************************************
	//
	kafkaPub    kafka.SyncProducer
	redis       *redis.Pool
	redisExpire int32
}

// New new a dao and return.
func New(c *conf.Config) *Dao {
	d := &Dao{
		c:           c,
		//
    	// ******************************************************************
	    // 下面这个 newKafkaPub(c.Kafka) 即是初始化 kafka
    	//  也就是连接上 kafka
    	//  下面, 我们先改写一下这个函数, 变通一下代码形式
    	//
    	// ******************************************************************
    	//
		kafkaPub:    newKafkaPub(c.Kafka),
		redis:       newRedis(c.Redis),
		redisExpire: int32(time.Duration(c.Redis.Expire) / time.Second),
	}
	return d
}

//  这是连接 kafka 的初化函数( function ) 
//  
func newKafkaPub(c *conf.Kafka) kafka.SyncProducer {
	kc := kafka.NewConfig()
	kc.Producer.RequiredAcks = kafka.WaitForAll // Wait for all in-sync replicas to ack the message
	kc.Producer.Retry.Max = 10                  // Retry up to 10 times to produce the message
	kc.Producer.Return.Successes = true
	pub, err := kafka.NewSyncProducer(c.Brokers, kc)
	if err != nil {
		panic(err)
	}
	return pub
}
```


这里, 先小改一下 func New(c *conf.Config) *Dao  这个函数
改成如下代码形式
```
// New new a dao and return.
func New(c *conf.Config) *Dao {
	d := &Dao{
		c:           c,
		//
		//
        // 注意, 下面这行被移出去
        // kafkaPub: newKafkaPub(c.Kafka),
        //
        //
		redis:       newRedis(c.Redis),
		redisExpire: int32(time.Duration(c.Redis.Expire) / time.Second),
	}
	//
    // 变成这样了, 功能没变化
    //
	d.kafkaPub = newKafkaPub(c.Kafka)
		
	return d
}
```

### 2.2 发布接口第二步, 检查一下哪个方法( method )需要被 interface 实现

还是看源代码
> 代码在 [https://github.com/Terry-Mao/goim/blob/master/internal/logic/dao/kafka.go](https://github.com/Terry-Mao/goim/blob/master/internal/logic/dao/kafka.go) 大约第13行开始

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
	b, err := proto.Marshal(pushMsg)
	if err != nil {
		return
	}
	
	//
	// ********************************
	//
	// 实际发布消息, 就是下面这个几行语句
	// 1. 组织一下需要发送的信息, 以 kafka 的发布接口要求的形式
	// 2. 尝试发布信息, 处理发布信息可能的错误
	//
	// 重点注意下面这几行, 后面会改掉
	// 重点注意下面这几行, 后面会改掉
	// 重点注意下面这几行, 后面会改掉
	//
	// ********************************
	//
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

// BroadcastRoomMsg push a message to databus.
func (d *Dao) BroadcastRoomMsg(c context.Context, op int32, room string, msg []byte) (err error) {
	pushMsg := &pb.PushMsg{
		Type:      pb.PushMsg_ROOM,
		Operation: op,
		Room:      room,
		Msg:       msg,
	}
	b, err := proto.Marshal(pushMsg)
	if err != nil {
		return
	}
	m := &sarama.ProducerMessage{
		Key:   sarama.StringEncoder(room),
		Topic: d.c.Kafka.Topic,
		Value: sarama.ByteEncoder(b),
	}
	//
	// ********************************
	// 实际发布消息, 就是下面这个语句
	// ********************************
	//
	if _, _, err = d.kafkaPub.SendMessage(m); err != nil {
		log.Errorf("PushMsg.send(broadcast_room pushMsg:%v) error(%v)", pushMsg, err)
	}
	return
}
```

-------------------

### 2.3 换用 interface 实现这个 SendMessage(m) 方法( method )

先上代码, 代码会说话( **golang 简单就在这里, 代码会说话** ) , 后加说明 

```

// PushMsg  interface for kafka / nats 
// ******************** 这里是新加的 interface 定义 *****************
type PushMsg interface {
	PublishMessage(topic, ackInbox string, key string, msg []byte) error  // ****** 这里小改了个方法名!!! 注意
	Close() error
}

// Dao dao.
type Dao struct {
	c           *conf.Config
	push        PushMsg   // ******************** 看这里 *****************
	redis       *redis.Pool
	redisExpire int32
}

// New new a dao and return.
func New(c *conf.Config) *Dao {

	d := &Dao{
		c:           c,
		redis:       newRedis(c.Redis),
		redisExpire: int32(time.Duration(c.Redis.Expire) / time.Second),
	}

	if c.UseNats {   // ******************** 在配置中加一个 bool 布尔值的开关项 *****************
		d.push = NewNats(c) // ******************** 这里支持 nats  *****************
	} else {
		d.push = NewKafka(c) //// ******************** 这里是原来的 kafka *****************
	}
	return d
}
```

------
kafka 实现 interface 接口的代码
```
// Dao dao.
type kafkaDao struct {
	c    *conf.Config
	push kafka.SyncProducer
}

// New new a dao and return.
func NewKafka(c *conf.Config) *kafkaDao {
	d := &kafkaDao{
		c:    c,
		push: newKafkaPub(c.Kafka),
	}
	return d
}

// PublishMessage  push message to kafka
func (d *kafkaDao) PublishMessage(topic, ackInbox string, key string, value []byte) error {

	m := &kafka.ProducerMessage{
		Key:   sarama.StringEncoder(key),
		Topic: d.c.Kafka.Topic,
		Value: sarama.ByteEncoder(value),
	}
	_, _, err := d.push.SendMessage(m)

	return err
}

```

---
nats 对 interface 的实现
```

// natsDao dao for nats
type natsDao struct {
	c    *conf.Config
	push *nats.Conn
}

// New new a dao and return.
func NewNats(c *conf.Config) *natsDao {

	conn, err := newNatsClient(c.Nats.Brokers, c.Nats.Topic, c.Nats.TopicID)
	if err != nil {
		return nil
	}
	d := &natsDao{
		c:    c,
		push: conn,
	}
	return d
}

// PublishMessage  push message to nats
func (d *natsDao) PublishMessage(topic, ackInbox string, key string, value []byte) error {
	if d.push == nil {
		return errors.New("nats error")
	}
	msg := &nats.Msg{Subject: topic, Reply: ackInbox, Data: value}
	return d.push.PublishMsg(msg)

}
```

----
最后, 调用 interface 的变更
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
	b, err := proto.Marshal(pushMsg)
	if err != nil {
		return
	}
	//
	// ********************************
	//
	// 实际发布消息, 就是下面这个几行语句
	// 1. 组织一下需要发送的信息, 以 kafka 的发布接口要求的形式
	// 2. 尝试发布信息, 处理发布信息可能的错误
	//
	// 重点注意下面这几行, 实际更改
	// 重点注意下面这几行, 实际更改
	// 重点注意下面这几行, 实际更改
	//
	// ********************************
	if err = d.push.PublishMessage(d.c.Kafka.Topic, d.c.Nats.AckInbox, keys[0], b); err != nil {
		log.Errorf("PushMsg.send(push pushMsg:%v) error(%v)", pushMsg, err)
	}
	return
}
```

OK, 修改完成

### 2.4 小结
#### 2.4.1 接口定义 (带命名的方法集合)
简明来说,  interface 接口定义一下名称, 再定义接口中要实现的方法 method ( 方法集合 )


```
// PushMsg  interface for kafka / nats 
// ******************** 这里是新加的 interface 定义 *****************
type PushMsg interface {
	PublishMessage(topic, ackInbox string, key string, msg []byte) error  // ****** 这里小改了个方法名!!! 注意
	Close() error
}

// Dao dao.
type Dao struct {
	c           *conf.Config
	push        PushMsg   // ******************** 看这里 *****************
	redis       *redis.Pool
	redisExpire int32
}
```

上面 定义了 PushMsg 这个interface , 这是一个 方法( method)集合

#### 2.4.2 方法定义与实现
1. 方法名 , 比如 PublishMessage 
2. input 数据, 就是这些 topic, ackInbox string, key string, msg []byte, 分别是
> 1. topic 这是 kafka 或 nats 里的主题, 也就是 pub/sub 发布/订阅的频道
> 2. ackInbox 这是 publish 发布的 confirm 确认频道
> 3. key 消息体( payload ) 的键
> 4. msg 这是消息体 payload 
3. ouput 数据, 这里是 error , 标示 PublishMessage 方法( method ) 的输出

这就是一个接口定义, 方法名/ 输入/ 输出, 至于方法的具体实现, 交由下面的实体去实现( 可以看 kafka / nats 中分别对应的 PublishMessage 的方法实现)

#### 2.4.3 接口实例化, 以便后面方法调用
很清楚, 方法是由具体实现来完成, 下面就是实例化方法

> 是用哪一个具体实现呢, 就看实例化哪一个了, interface 最终落地, 就在这里

```
	if c.UseNats {   // ******************** 在配置中加一个 bool 布尔值的开关项 *****************
		d.push = NewNats(c) // ******************** 这里支持 nats  *****************
	} else {
		d.push = NewKafka(c) //// ******************** 这里是原来的 kafka *****************
	}
```

而在 func (d *Dao) PushMsg(c context.Context, op int32, server string, keys []string, msg []byte) (err error) 中, 则简单调用 interface 定义的方法

#### 2.4.4 接口方法调用 
与其他方法 method 或函数 function 是一样的, 没什么特别的

```
	// ********************************
	if err = d.push.PublishMessage(d.c.Kafka.Topic, d.c.Nats.AckInbox, keys[0], b); err != nil {
		log.Errorf("PushMsg.send(push pushMsg:%v) error(%v)", pushMsg, err)
	}
```


### 3. 浅谈 golang 的 interface --> 解耦合!!

再一次回看,

在 [吴德宝AllenWu](https://juejin.im/post/5a6873fd518825734501b3c5) 文章[Golang interface接口深入理解](https://juejin.im/post/5a6873fd518825734501b3c5) 中这样写到:


> 为什么要用接口呢？在Gopher China 上的分享中，有大神给出了下面的理由：
>
> > writing generic algorithm （类似泛型编程）
> >
> > hiding implementation detail （隐藏具体实现）
> >
> > providing interception points (提供拦截点-----> 也可称叫提供 HOOKS , 一个插入其他业务逻辑的钩子)


interface 确是**隐藏了具体实现**, 能让我们很容易的把 goim 对 kafka 的依赖, 切换到 nats , 并且通过一个开关项, 来确定使用哪一个具体实现

扩展一下, 这个 interface 也可以实现从 kafka 切换到 rabbitMQ / activeMQ / redis (pub/sub) ....
只要简单实现 PushMsg 这个 interface 就好啦

### 4. 源代码及其他补充 

另有 goim 在 job 网元上的 subscribe 订阅接口, 支持 interface 代码是一路子方法, 直接看源码吧, 有交流讨论再另写.

> 注: job 代码中, 我把某个方法( method ) 拆解成了函数( function ), 有兴趣的朋友可以查一下, 有些小区别,但效果一样. 


goim 源代码在[https://github.com/Terry-Mao/goim](https://github.com/Terry-Mao/goim)

示例代码在[https://github.com/tsingson/goim](https://github.com/tsingson/goim)



### 5. 扩展, 看看 gRPC 中的解耦合

gRPC , 就是 google 的 RPC  ( Remote Procedure Call) , 看一下 gRPC 以 go 实现的 interface 定义

#### 5.1 先看原始的 protobuf 定义

>
> **protobuf 是 gRPC 中默认的 接口定义, 就像 爱立信 ICE ( 开源版本是 zeroICE ) 的 clice , apache 的 thrift**
>


在 goim 中, 网元间用 gRPC 通讯, 再看图

![](https://user-gold-cdn.xitu.io/2019/4/22/16a43da4dc840cdb?w=1328&h=790&f=png&s=92400)
看图上的 grpc 标示, 注意, 图上标示箭头不完全准确:

grpc 同时支持
> * 普通 Client / Server 调用(北向)接口
> * Client 向 Server 的流式(北向)流式接口
> * Server 向 Cinet 调用(南向)流式接口
> * 以及 Server / Client 双向流式接口

网上文章很多, 不一一展开了. 我们重点关注一下, golang 中对 gRPC 的实现, 也就是 golang 如何把 protobuf 定义的接口, 定义为 golang 中的 interface , 以及如何具体实现 interface .

----
**看码, 看码, 看码:**


> 源码在[https://github.com/Terry-Mao/goim/blob/master/api/comet/grpc/api.proto](https://github.com/Terry-Mao/goim/blob/master/api/comet/grpc/api.proto)

```
syntax = "proto3";

package goim.comet;
option go_package = "grpc";

//......
//
// ************************
// 这里定义 input 输入

message PushMsgReq {
    repeated string keys = 1;
    int32 protoOp = 3;
    Proto proto = 2;
}
//
// ************************
// 这里定义 output 输出 
message PushMsgReply {}

//.........

service Comet { 
    // ..........
    //PushMsg push by key or mid
    //
    // ************************
    // 这里定义接口, 这个接口可以由
    // golang / java / rust / js / python / php ...实现
    //
    // 这是解耦合的极致啊!!!!!!!!!!!!!!!!
    //
    // ************************
    //
    rpc PushMsg(PushMsgReq) returns (PushMsgReply);
    // Broadcast send to every enrity
    // ...........
}

```
#### 5.2 gRPC 中 go 实现的 interface 定义

注意, 下面的源码是 protobuf 自动生成的, 不需要编辑更改, 注释是方便沟通额外加的


> 源码在 [https://github.com/Terry-Mao/goim/blob/master/api/comet/grpc/api.pb.go](https://github.com/Terry-Mao/goim/blob/master/api/comet/grpc/api.pb.go)


```
// Server API for Comet service
// ************************
// 这里定义接口, golang 实现服务器端
// ************************
    
type CometServer interface {
    ...
	// PushMsg push by key or mid
	//
    // ************************
    // 这里定义接口, golang 的接口中的方法
    // ************************
    //
	PushMsg(context.Context, *PushMsgReq) (*PushMsgReply, error)
    ...
}
```

#### 5.3 gRPC 中 go 实现的 interface 实例化

最后, 具体实例化代码实现, 在

[https://github.com/Terry-Mao/goim/blob/master/internal/comet/grpc/server.go](https://github.com/Terry-Mao/goim/blob/master/internal/comet/grpc/server.go) 

代码会说话儿, 这里就不展示了.

 






### 6. 郑重警告

谢谢朋友们看到最后, 写码挣钱的朋友都是有一说一, 这里声明一下:

**代码中把 kafka 写成可用 nats 替换, 只是技术上的学习与尝试, 并不是建议或推荐使用 nats:**

* nats 并不保障消息送达
* nats 并不提供持久化
* nats 用在 goim 上的效率, 还需要压测

所以, case by case , 具体业务场景具体分析, 商用项目的选型, 是一个慎重而严谨的事儿

请自行评估风险/成本

.

.


感谢 [https://www.bilibili.com](https://www.bilibili.com) & [毛剑](https://github.com/Terry-Mao) 及众多开源社区的朋友们


欢迎交流与批评.....
.

.



### 关于我
网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  




 [tsingson](https://github.com/tsingson) 写于中国深圳 小罗号口琴音乐中心,   2019/04/22