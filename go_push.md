Date: 2015-10-31
Title: gopush 折腾记录
Tags:  go 推送 
Toc:no
Status: public
Position: 1

最近公司用的推送,极光和Leancloud,都有不稳定的情况.而且聊天服务用的也是Leancloud,也是十分不稳定.自己于是研究了几天Golang,打算写一个推送.老大也着急了,准备这几天必须搞定,采用之前猎豹的剑总的推送方案,于是搭起来研究一下.

毛剑的Gopush是完全基于Golang的,第一版的单机的Gopush还是很简陋的,后面的Gopush-Cluster支持了Zookeeper与集群,性能非常不错,单机大概能撑80w左右的连接,之前在猎豹的时候也听过分享.代码大概也看了看,觉得跑起来应该没什么难度的

项目地址:[GitHub](https://github.com/Terry-Mao/gopush-cluster)

先搞环境.找到的教程是1.4版本的下载地址(然而毛总那个是基于1.3的,我觉得应该没问题):
```
wget https://storage.googleapis.com/golang/go1.4.2.linux-amd64.tar.gz 

sudo tar -C /usr/local -xzf go1.4.2.linux-amd64.tar.gz
```

设置环境变量

```
export GOPATH=/opt/src/go
export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin 
``` 

然后是依赖问题.服务器没法翻墙,其中的websocket依赖又是在code.google.com域名下,拉不下来.无奈上自己的vps,又配一次环境,手动拉一下
```
go get code.google.com/p/go.net/websocket 
```
然后到相应的文件夹打个包发到服务器就ok了.其实一开始我从git上找到了一个这个包的clone,后面遇到问题才又这么折腾了一下,然而并不是这块的问题

依赖都搞定 ,按着教程就可以跑起来了.测试一下 

```
curl -d "{\"m\":\"{\\\"test\\\":1}\",\"k\":\"t1,t2,t3\"}" http://localhost:8091/1/admin/push/mprivate?expire=600
``` 
可耻的失败了,返回{"ret":65535}.按着wiki里写的,这是内部错误啊...根据.xml的配置里面的错误日志,发现错误来自Redis部分,错误大概是
```
Redis.dial("","")  unknow network
``` 
说明redis的配置解析有问题.这时候就体现出开源的好处了,拿出源码开搞吧 

kill掉message模块,直接在message目录下 go run *.go -c /opt/src/go/bin/message.conf 就可以启动了,在需要的地方print很方便

```
grep 'redis.Dial' -rl .
```
发现这个错误是在 message/Redis.go文件中,配置是通过这个正则解析的:

```
reg := regexp.MustCompile("(.+)@(.+)#(.+)|(.+)@(.+)")
```
解析后的数据在pw这个slice里,然而访问pw[1]和pw[2]并没有值,却在pw[5]和pw[6]里.修改正则为
```
reg := regexp.MustCompile("(.+)@(.+)")
```
后问题解决,原来的正则是匹配带密码情况的,估计他们测试的时候一直使用的带密码的redis,然而线上我们一般redis只监听localhost所以不用密码.这个解决掉就没有别的问题了(在此感谢亮哥大牛帮忙调试),顺利跑起

服务器跑起来了,Client又是问题.拿Js的SDK去实验,总是不成功.拿着源码去调试,根本没有进入到负责websocket handleFunc的那个sub方法中,可能是websocket版本的问题?改天继续解决...




