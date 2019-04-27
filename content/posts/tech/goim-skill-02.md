---
title: "bilibili/discovery 基本概念及在 goim中的使用"
date: 2019-04-24T22:02:57+08:00
hidden: false
draft: false
tags: [golang, go, goim]
keywords: [tsingson]
description: "bilibili/discovery 基本概念及在 goim中的使用"
slug: "goim-go-03"
---


## 0. 背景与动机

在学习 goim 过程中, [bilibili/discovery](https://github.com/bilibili/discovery) 是一个服务注册/发现的依赖网元, golang 实现了 [netflix/eureka](https://github.com/Netflix/eureka) 并作了一些扩展改进


这里顺带记录了对  [bilibili/discovery](https://github.com/bilibili/discovery) 学习过程中的一些理解


## 1. discovery 在goim 中的角色与作用

![](https://user-gold-cdn.xitu.io/2019/4/25/16a522a08fd354cd?w=1329&h=831&f=png&s=92443)

上图标示了  [bilibili/discovery](https://github.com/bilibili/discovery) 在 [goim](https://github.com/Terry-Mao/goim) 中的位置, 与作用(以 comet / job 为例):

 

![](https://user-gold-cdn.xitu.io/2019/4/25/16a531eb473f3ba8?w=1079&h=619&f=png&s=60681)

* 部署一到多分 discovery 作为服务注册/发现网元 ( discovery 相互间会同步注册数据,细节见后)
* comet 一到多个部署, 这里是一个 comet gRPC server 服务端
    * comet 启动果, 每一个部署向 discovery 进行--> **服务注册**
    * 注册成功后与 discovery 之间保持一个健康状态同步( renew ), 见标示 1
    * comet 如果下线, discovery 会标示下线状态
* job 一到多个部署, 这里是一个 comet gRPC client 客户端
    *  job 启动后, 向 discovery 进行 polls 获取 goim-comet 所有服务实例列表--> **服务发现**
    *  job 持续监听 discovery 中的 goim-conet 服务节点列表, 同步到本地
    *  job 向 goim-comet 实例( 整个列表) 分发 goim 消息 ---> **job 的主体业务功能**

如果 discovery 网元不存在, 那很简单, job 在配置文件中写死 comet 地址( 一到多个), job 的 comet-gRPC-client 直接向 comet 的 comet-gRPC-server 进行互通完成业务. 这样就失去了分布式的动态扩展能力

----
discovery 之间, 会同步注册的服务实例信息

**注意**
在 bilibili/discovery 中, discovery 本身被标记为
 ```
_appid = "infra.discovery"
 ```
相互之间一样进行相互注册/更新, 同在相互之间同步 名为"infra.discovery" 与其他 app 的实例信息

在 discovery 的配置文件中, discovery 实例被称为 node , 由 nodes 参数进行配置, 配置定义如下
```
// Config discovery configures.
type Config struct {
	Nodes  []string  # ******************** 这是配置一到多个 discovery 实例的定义
	Region string
	Zone   string
	Env    string
	Host   string
}
```



## 2. discovery / eureka 的基本概念

### 2.1 基本概念

![](https://user-gold-cdn.xitu.io/2019/4/25/16a531f0be908f57?w=1079&h=703&f=png&s=125303)

discovery / eureka 中的基本概念, 如上图所示, 就是一个分区进行注册/调度的简单划分
* Region 地区, 例如, 中国区, 南美区, 北美区...
* Zone 可用区域, 例如中国区下的 gd 广东地区, sh 上海地区, 一般是指骨干 IDC 机房, 或者跨地区的逻辑区域, 这是同区内调度的主要划分点. 一般是同区内调度, 不会跨区调度
* Env 再划分小一点的运行环境划分, 比如 Env = dev 开发环境, Env = trial 试商用...
* appID 这是注册应用的名称, 服务注册与发现, 依赖的是 name ----> address 名称到地址的注册(写入/更新) 与发现( 获取名称对应的服务地址或服务地址列表)

-------

> 
> **注**: bilibili/discovery 是以 http 方式提供注册/更新/发现/同步...等服务注册与发现等业务功能
> 

所以, 可以看到 discorevy 获取一个服务器节点, 是如下方式
```
curl 'http://127.0.0.1:7171/discovery/fetch?zone=gd&env=dev&appid=goim.comet&status=1'
```
上面 URL 中, zone 对应就是获取 gd 广东区域内, 环境定义为 dev , appID 为 goim.comet 的服务器实例, 当然, status 是附加约束, 这里 status=1 表示过滤名称为 goim.comet 的服务器实例状态要求为 status = 1 ( 即接收服务请求的 goim.comet 实例**列表**)

上面的 curl 会返回以下结果
```
{
    "code": 0,
    "data": {
        "instances": {
            "gd": [
                {
                    "zone": "gd",                # *** ** 可用区域
                    "env": "dev",                # ****** 运行环境
                    "appid": "goim.comet",       # ****** appID 名称
                    "hostname": "hostname000000",
                    "version": "111",
                    "metadata": {
                        "provider": "",
                        "weight": "10"
                    },
                    "addrs": [
                        "http://172.1.1.1:8080",
                        "gorpc://172.1.1.1:8089" # ****** 有效的 gRPC 地址
                    ],
                    "status": 1,
                    "reg_timestamp": 1525948301833084700,
                    "up_timestamp": 1525948301833084700,
                    "renew_timestamp": 1525949202959821300,
                    "dirty_timestamp": 1525948301848680000,
                    "latest_timestamp": 1525948301833084700
                }
            ]
        },
        "latest_timestamp": 1525948301833084700
    }
}
```

discovery/ eureka 换成 DNS 域名 可以在逻辑上表示为 schema://appID.Env.Zone.Region , 类似于 grpc://goim.comet.dev.gd.china.xxxxx.com

换成 etcd 可以表示为 /Region/Zone/Env/appID, 例如  "/china/gd/dev/goim.comet" 

### 2.2 小结与配置建议
由上小节可知, bilibili/discovery 或 netflix/eureka 的配置中, 以下4个关键参数, 需要一一对应
* region
* zone
* env 或 deployEnv 
* appID

在 goim 中, appID 已经在代码中标记为常量, 如下
```
# github.com/Terry-Mao/goim/cmd/comet/main.go

const (
	ver   = "2.0.0"
	appid = "goim.comet"
)

# github.com/Terry-Mao/goim/cmd/logic/main.go
const (
	ver   = "2.0.0"
	appid = "goim.logic"
)

```




## 3. goim 中使用 bilibili/discovery

还是以 logic / comet 之间的 gRPC 为例

所有使用 bilibili/discovery 的配置是类似的, 在配置中, 包含以下定义

> 原始定义在 [https://github.com/bilibili/discovery/blob/master/naming/client.go](https://github.com/bilibili/discovery/blob/master/naming/client.go) 第 46行开始

```
// Config discovery configures.
type Config struct {
	Nodes  []string  # ******************** 这是配置一到多个 discovery 实例的定义
	Region string
	Zone   string
	Env    string
	Host   string
}
```

在 comet 配置中定义为

> 
> 在 comet 配置源文件中 [https://github.com/Terry-Mao/goim/blob/master/internal/comet/conf/conf.go](https://github.com/Terry-Mao/goim/blob/master/internal/comet/conf/conf.go) 第 112 行
> 

```
// Config is comet config.
type Config struct {
	Debug     bool
	Env       *Env            # ******************** 这里这里这里
	Discovery *naming.Config  # ******************** 这里这里这里
	TCP       *TCP
	Websocket *Websocket
	Protocol  *Protocol
	Bucket    *Bucket
	RPCClient *RPCClient
	RPCServer *RPCServer
	Whitelist *Whitelist
}
```

> 
> 在 job 配置源文件中 [https://github.com/Terry-Mao/goim/blob/master/internal/job/conf/conf.go](https://github.com/Terry-Mao/goim/blob/master/internal/job/conf/conf.go) 第 59 行

```
// Config is job config.
type Config struct {
	Env       *Env            # ******************** 这里这里这里
	Kafka     *Kafka
	Discovery *naming.Config  # ******************** 这里这里这里
	Comet     *Comet
	Room      *Room
}
```

就像第二节所说的, regoin / zone / env ,  所以, 重点关注 Env / Discovery 两个配置定义, 重点在 Discovery 配置naming.Config 即可

### 3.1 在 comet 中的服务注册, 与服务更新

#### 3.1.1 注册如下
> 源代码见 [https://github.com/Terry-Mao/goim/blob/master/cmd/comet/main.go](https://github.com/Terry-Mao/goim/blob/master/cmd/comet/main.go) 第42/43 行

```
	// register discovery
	dis := naming.New(conf.Conf.Discovery)
	resolver.Register(dis)
	
```
#### 3.1.2 更新如下

该 comet 的注册信息更新代码放在一个 goroutine 中, 每10秒更新一次

> 源代码见 [https://github.com/Terry-Mao/goim/blob/master/cmd/comet/main.go](https://github.com/Terry-Mao/goim/blob/master/cmd/comet/main.go) 第42/43 行

```
			if err = dis.Set(ins); err != nil {
				log.Errorf("dis.Set(%+v) error(%v)", ins, err)
				time.Sleep(time.Second)
				continue
			}
			time.Sleep(time.Second * 10)
```


 ### 3.2 在 job 中的服务发现

 #### 3.2.1 job 中的注册代码, 实际是无用代码
 在 job 代码中, 含有服务注册代码, 实际上是无用代码, 原因是, 只有服务端才需要进行服务注册, 而 job 实际上只有两个业务关联逻辑
 1. 对 kafka 进行消息订阅
 2. 向 tomet 中的 comet gRPC server 进行消息 push 推送

> 代码在 [https://github.com/Terry-Mao/goim/blob/master/cmd/job/main.go](https://github.com/Terry-Mao/goim/blob/master/cmd/job/main.go) 第 28行

```
	// grpc register naming
	dis := naming.New(conf.Conf.Discovery)
	resolver.Register(dis)
```

#### 3.2.2 job 中的服务发现代码

> 代码在 [https://github.com/Terry-Mao/goim/blob/master/internal/job/job.go](https://github.com/Terry-Mao/goim/blob/master/internal/job/job.go) 第 85行
```

func (j *Job) watchComet(c *naming.Config) {
	dis := naming.New(c)       # **************************** 构造符合 gRPC 要求的服务发现实例
	resolver := dis.Build("goim.comet")
	event := resolver.Watch()  # **************************** 监听 服务发现, 这里返回一个 channel
	select {                   # **************************** 从 channel 中循环获取返回
	case _, ok := <-event:
		if !ok {
			panic("watchComet init failed")
		}
		if ins, ok := resolver.Fetch(); ok {   # **************************** ins 即是返回的实例
			if err := j.newAddress(ins.Instances); err != nil {
				panic(err)
			}
			log.Infof("watchComet init newAddress:%+v", ins)
		}
	case <-time.After(10 * time.Second):
		log.Error("watchComet init instances timeout")
	}
	go func() {
		for {
			if _, ok := <-event; !ok {
				log.Info("watchComet exit")
				return
			}
			ins, ok := resolver.Fetch()     # **************************** ins 即是返回的实例
			if ok {
				if err := j.newAddress(ins.Instances); err != nil {
					log.Errorf("watchComet newAddress(%+v) error(%+v)", ins, err)
					continue
				}
				log.Infof("watchComet change newAddress:%+v", ins)
			}
		}
	}()
}
```

## 4. bilibili/discovery 架构与实现简要解读

.............稍后一一道来, 哈, 先去挣点钱先.............

.

.

.


欢迎交流与批评.....
.

.



关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  



[tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://zhuanlan.zhihu.com/tsingsonqin), 2019/04/25