## p2p的架构设计及部分实现细节
</br>

#### 一 前言
##### 1 工作方式
p2p模块总是在区块链的众多子模块中优先启动，它承担将本地消息向外发送，获取外部消息的职责；获取与发送消息之前自身又需要维持一个最低的链接数，维持这个最低链接数就需要获取跟多节点信息。
##### 2 本文介绍
与区块链的多个子模块交互、消息传输的隐私性与安全性、去中心化等一系列问题，使得设计一个p2p模块自然会涉及到很多问题， 然而很多优秀的公链已经为我们提供了一些参考的实例。 本文分析主流公链的的p2p模块及实现的一些细节问题，但是限于篇幅的原因仅会给出部分具体实例。
</br>

####  二 架构分析及实现细节
   从职责划分上来讲， p2p模块可分为两块：_发现服务_、_消息传输服务_。发现服务主要用来获取更多节点信息，使得消息传输服务在与其他节点建立链接时有更多的选择， 节点信息包括IP、端口、协议、节点ID等，链接到更多的节点意味着在以下方面有更大概率：更新的本地状态、同等算力下更多的节点收到本地挖出的block。消息传输服务，顾名思义，就是用来传输消息的，比如共识消息、状态同步消息、心跳包、tx/block广播消息等。 消息传输服务从业务深度上可划分为：
   + 链接资源申请管理
   + 协议/代码版本检测
   + 隐私保护
   + 消息路由
   + 链接资源释放管理

##### 1 发现服务
主流公链的发现服务一般有三个组成部分： 
  + 节点信息的持久化与查询
  + 引擎部分：维护链接池、调度节点发现
  + 节点发现， 主要使用pex、dht算法，概括的来讲:
	- pex是在已建立链接的俩个节点之间交换各自已知节点信息
	- dht解决了p2p网络中心化tracker的问题

###### 1.1 节点信息与链接池维护
节点信息的应该区分为：
+ 已经建立过链接的节点
+ 未建立过链接的节点，通过节点发现获得

未建立过链接的节点应该额外的保存两条信息
 + 最后一次尝试链接的时间, 用于控制尝试链接的间隔时间 
 + 尝试的次数，次数超过一定数量，该信息应该被删除
 
重启后应该优先尝试已经建立链接的节点。 go-ipfs采用bitswap算法， 该算法通过负债率r调整消息发送概率P， 负债率越大消息被丢弃的概率越大。通过累计发送和接受到的消息字节数计算负债率， 进一步调整远程节点的消息被处理的概率，所以我们也建议将这这俩个指标持久化， 供消息模块使用以及尝试链接的优先级。尝试失败超过一定的数量后， 可以认为这条信息已经失效了。

```golang
计算负债率、丢弃概率公式如下：
r = bytes_sent / (bytes_recv + 1) 
P = 1− 1/(1+exp(6−3r))
```
###### 1.2 节点发现
pex算法本身很简单， 我们不做的过多的赘述。kademlia作为目前dht的主流实现，其算法主体思想如下：

    1. A节点广播一个（hash， entry）的键值对， 一般为(节点ID, NetworkAddr)
    2. B将(hash, entry)加入本地table 
    3. B使用table提供基于距离关系的查找

table的结构为[距离]list， len(list) < 20（建议）， 将(hash, entry)加入table的大致过程如下， 可以一定程度的抵抗女巫攻击
```golang
idx := 计算与本节点ID的距离
table[idx] = append(table[idx], entry)
backup(table[20:]) // 作为备份信息
table[idx] = table[idx][:20]
```
计算距离的算法如下:
```golang
func prefixLength(xor uint8) uint {
	switch {
	case xor == 1:
		return 7
	case xor >= 2 && xor <= 3:
		return 6
	case xor >= 4 && xor <= 7:
		return 5
	case xor >= 8 && xor <= 15:
		return 4
	case xor >= 16 && xor <= 31:
		return 3
	case xor >= 32 && xor <= 63:
		return 2
	case xor >= 64 && xor <= 127:
		return 1
	case xor >= 128 && xor <= 255:
		return 0
	}
	return 8
}

func calcDistance(a, b []byte) uint {
	c := uint(0)
	for i := 0; i < len(a) && i < len(b); i++ {
		x := a[i] ^ b[i]
		if x == 0 {
			c += 8
		} else {
			c += prefixLength(x)
			break
		}
	}
	return uint(len(a))*8 - c

}
```

</br>
    
##### 2 消息传输服务
###### 2.1 链接资源申请管理
p2p的消息传输服务应该不仅作为server通过监听端口处理链接请求， 还需作为client发起链接， 俩个方向的链接都应通过令牌的方式控制连接数。仅作为server处理链接会降低攻击难度；仅作为client发起链接会失去大量链接的机会，因为其他节点发起与本节点的链接失败后会抛弃本节点信息。此过程有俩个细节需要说明：
+ 节点之间保持一个链接即可，不要漏过处于dialing状态的节点
+ 需要标识是链接的发起方还是监听方，原因是:
  - 后续的心跳等模块可能需要区分， 发起方作为client需要维持心跳
  - 释放链接是需要依据标识归还不同的令牌

为了保证消息的完整性， ethereum使用如下的消息报文格式来拆解包
```golang
| length| msgHash | msg |
```

###### 2.2 协议/代码版本检测
在具体的业务逻辑开始前进行本代码版本检测可以实现控制代码升级、分叉，进行协议检测可以避免大量无效链接。目前ethereum的节点分为俩种full node、light node, 即将到来的sharding版本又增加了更多的节点角色类型， light node仅同步header信息以及向full node请求merkel proof， 而full node是否支持为light node提供服务就需要在这一步确认。 

###### 2.3 隐私保护
区块链建立在非对称加密算法ECC上，非对称加密算法做加解密效率很低，此外ECC是无法用来加解密信息只是用来’验证‘ ， 所以提供消息隐私性功能需要借助对称加密算法比如AES算法。一般使用STS实现AES密钥同步， 算法如下

```golang
concat := func(a, b uint)uint{ 
     v, _ := strconv.Atoi(fmt.Spintf("%d%d", a, b))
     return uint(v)
} 
mod := func(x, y uint)uint{
    return x % y
}
pow := func(x, y uint)uint{
  return uint(math.Pow(x, y))
}
算法：
A, B : set p = 23, g = 5 
A    : generate a = 6,  set va=mod(concat(g, a), p)=8 , sendTo(B,  8) // 56 % 23 == 8 
B    : generate b = 15,  set vb=mod(concat(g, b), p)=19 , sendTo(A,  19) // 515 % 23 == 19 
A    : set secret=mod(pow(vb, a), p)=2 // 47045881 % 23=19
B    : set secret=mod(pow(va, b), p)=2 // 35184372088832% 23 = 2

密钥安全性保证证明如下:
  +----------节点A-------------+-------节点B----------+---恶意节点 -----------+
  +                           +                      +                      + 
  已知信息  a,p,g,concat,mod   +  b, p,g,concat,mod   +  p,g,concat,mod      +   
  +                           +                      +                      + 
  已知信息  vb, a, p, pow, mod +  va, b, p, pow, mod  +  va,vb,p,pow,mod     + 
  +                           +                      +                      +
  已知信息    secret           +   secret             +         -            +
  +-------------------------------------------------------------------------+
```
###### 2.4 消息路由
主流的公链包括若干个模块比如：共识、交易池、账本、P2P等。 共识模块需要通过P2P 广播投票信息、收集投票出块；交易池将验证tx通过p2p 广播给周围节点， 通过p2p收集tx验证后加入本地交易池；账本通过p2p同步账本信息，比如我们可以在心跳包中加入账本高度， 账本模块对比本地高度请求block。
ethereum的交易池模块业务流程如下：
+ 接收tx （来源包括rpc调用、p2p转发等）
+ tx验证（包括nonce， value、gas、signature）后加入本地交易池， 并依据tx中的nonce删除老旧tx
+ 通过gasprice对交易池里的tx排序 
+ 为出块模块（挖矿、共识）提供tx list
+ 调用p2p广播验证过的tx 
+ 监听eventhub的“挖矿”主题，删除本地交易池中已经被出块模块打包进入block的tx
+ 监听eventhub的”新块“主题，删除本地交易池中已经其他节点打包进入block的tx
我们可以清晰的看到p2p与交易池的交互发生：1、5。交易池与p2p调用实现大致如下
```golang
type Message struct{
   topic string    // 消息主题， 此处是 txpool
   code  uint       
   payload []byte  // 消息体， 解析方式需由各模块自定义 推荐google/proto.Marshal/Unmarshal
}
type Context interface{
	Send(uint32, []byte)  // 模块调用p2p向远程节点发送消息
	
	ID() string           //  节点标识符
	Topic()string	
   ...
}

type PeerHandler interface {
	NewPeer(Context)TopicHandler
   ...
}

type TopicHandler interface {
	Setup()error
	Handle(uint32, []byte) error // 远程节点向本节点发消息
        Treardown()
}

func readLoop(conn Conn){
   for{
      data := conn.Read() // 消息完整性检测、数据加解密在conn封装层完成
      go disptath(data)
      ......
   }
}

var mTopicHandler map[string]TopicHandler

func dispatch(data []byte){
   topic, code, payload := decode(data) 
   h, exists := mTopicHandler[topic]  // 依据 topic做路由 
   if exists{
      h.Handle(code, payload)
   }
   ......
}

type TxPoolServer server{
   txHandlers map[string]txHandler
}

func (self *TxPoolServer)NewPeer(ctx context)TxPoolHandler{
   h := txHandler{...}
   self.txHandlers[ctx.ID()] = h
   ......
   return h
}

func (self TxPoolServer)broadcastTx(tx Transaction， except func(string)bool){ // 广播交易
   payload = encode(tx)
   for id, h := range self.txHandlers{
      if except!=nil && except(id){  // 哪些节点不能发
         continue
      } 
      h.Send(1, payload)
   }
}

func (self txHandler)Handle(code uint, msg []byte)error{
   switch code{
   case 1: // new tx
      ......
   }
   return nil
} 

```
p2p模块应该对外只承担消息发送与接受、对内实现节点发现的功能， 具体的业务逻辑不应该交由p2p处理，事实上主流公链也都采用如下的架构来解耦具体业务：
 + p2p模块负责网络读
 + conn 封装
 + 消息解析及路由
 + handler处理业务
 + 消息拼装
 + p2p模块负责网络写

###### 2.5 链接资源释放管理
链接的client端应该负责发送心跳包维持链接， server端释放不活跃、异常的client; 释放链接的同时应该同步通知与该节点交互的各个子模块。注意前文提到需要‘归还不同的令牌‘。
</br>

#### 3 总结
以上只是简单描述了设计几个核心模块时需要主要注意的问题以及几条建议，当然还有一些其他问题比如：流量控制、节点管理、ssl transport、Conn封装、消息去重等，这些问题之所以没有展开是因为它们不参与核心流程， 但是如果设计的时候没有考虑到，那么这个p2p模块也只能算是基本可用。

