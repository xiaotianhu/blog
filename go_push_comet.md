Date: 2015-10-31
Title: gopush代码分析-comet模块
Tags:  go 推送 
Toc:no
Status: public
Position: 1

===Comet模块===
>负责SOCKET连接处理,支持TCP SOCKET与WEBSOCKET两种协议,数据传输采用REDIS的序列化协议

main启动后,初始化如下
InitConfig()
perf.Init(Conf.PprofBind)
NewChannelList()
StartStats()
StartRPC()
StartComet()
InitZK()
InitSignal()

其中,NewChannelList()根据 Conf.ChannelBucket指定的数量,初始化一定量的 &ChannelBucket对象,并把这些bucket加入到&ChannelList中.整个ChannelList的结构很简单:
```
type ChannelList struct {
	Channels []*ChannelBucket
}
```
Bucket可以理解为一个池,初始化好这个池后,每一个新进来的链接,都会根据MruMruHash算法,把SeqChannel平均分配加入到Bucket这个池子中,这个后面会讲到.

先从主体开始说起吧,StartComet()当然是主要的了,根据配置里的proto协议,分别或者全部启动TCP与WEBSOCKET监听,此处拿TCP监听为例.StartTCP()后,根据Conf.TCPBind的配置,启动多个TCP监听routine:

```
go tcpListen(bind)
```

开始Listen后,在for循环中不停的开始Accept,并且设置了TCP相应的参数,每个新进入的连接,开启一个新的routine处理:
```
for{
    conn.SetKeepAlive(Conf.TCPKeepalive)
    conn.SetReadBuffer(Conf.RcvbufSize)
    conn.SetWriteBuffer(Conf.SndbufSize)
    conn.SetReadDeadline(time.Now().Add(fitstPacketTimedoutSec))
    go handleTCPConn(conn, rc)
}
```

进入处理SOCKET数据的阶段了,BufIoReader读取SOCKET的数据,解析参数 

```
parseCmd(rd)
```

值得一说的是GoPush采用的(REDIS的协议)[http://redis.io/topics/protocol],这篇说明写的很详细了.就是一个类似JSON的序列化协议,支持数组/字符串/数字/错误信息/几种简单数据结构的传递,比较友好明文可读,并且解析效率高,堪比2进制~

args[0]代表命令,默认必须是sub,否则报错.这块的逻辑跟Redis的PUB/SUB模型比较类似,都是Client订阅某个Channel,然后等待消息推送过来.
接下来对args中的几个参数,key/heartbeat/token/version进行判断,然后新建一个用于通讯的channel:

```
UserChannel.Get(key, true)
```

这个UserChannel.Get()方法又做了一些事 比较复杂.之前在初始化的时候,新建了一个ChannelList,里面保存了一组初始化好的Bucket,用来做Channel的池.Get方法,先验证一下Client需要Sub的Key是不是属于这个Comet节点的:

```
node := CometRing.Hash(key)
if Conf.ZookeeperCometNode != node {
	log.Warn("user_key:\"%s\" node:%s not match this node:%s", key, node, Conf.ZookeeperCometNode)
	return ErrChannelKey
}

```

CometRing.Hash 就是采用ketama(ketama:libketama-style consistent hashing in Go)的一致性哈希算法,[ketama项目地址](https://github.com/mncaudill/ketama)
验证通过后,从ChannelList的Bucket中,获取这个Key对应的SeqChannel,如果没有则新建

```
b := l.Bucket(key)
if c, ok := b.Data[key]; !ok {
    if !Conf.Auth && newOne {
        c = NewSeqChannel()
        b.Data[key] = c
        ...
        return c, nil
    } else {
      ...		
    }
} else {
    ...
    return c, nil
}
```

SeqChannel是外部与Client沟通的通道,比如从Web发过来的消息,通过RPC传给Comet模块,在通过SeqChannel发给Client,另外还有Token验证的功能,比较简单一看就好了

接下来就是在这个routine中持续监听Socket数据流并发送心跳了,如果产生错误就break并删除SeqChannel.

```
for {
    conn.SetReadDeadline(time.Now().Add(time.Second * time.Duration(heartbeat)))
    conn.Read(reply)    
    if string(reply) == Heartbeat {
        ...
        conn.Write(HeartbeatReply) //服务器发来心跳,并写一个心跳回去
    }
    end = time.Now().UnixNano() //更新SOCKET超时时间
}
```

>到此,主要的Socket监听/通讯/心跳就完成了.













