+++
title = '上线Blog聊天室'
date = 2024-01-31T00:20:00+08:00
+++

本来是一个平平无奇的一天

但是由于准备把之前写完的聊天室上线就有了今天的这篇Blog

架构介绍：

用Gin起了个单体+Kratos的数据库操作端

- [Lan-chat](https://github.com/Sikyy/Lan-chat)


- [Kratos-Redis-Chat-Demo](https://github.com/Sikyy/Kratos-Redis-Chat-Demo)

一开始我是没想着做数据库的，后来发现需要检测一下在线人数，刚好那段时间在写kratos和Redis就直接在那上面写了，于是就有了算是两个端的服务

于是我打算这么做，kratos自带的dockerfile，我就直接把它打包成docker+把Lan-chat直接移植到服务器上，启动，这就很简单的事情嘛，配个证书，配个反代的事情。正当我这么想着的时候，事情才刚刚开始

首先是Kratos的Docker部分


打包：

参考Kratos的文档：[https://go-kratos.dev/docs/devops/docker](https://go-kratos.dev/docs/devops/docker)

基本是那个没什么需要改的地方只需要把

```docker
CMD ["./server", "-conf", "/data/conf"]
```

里面的server改成你自己项目的文件名就好了，ok，开始构建 

由于需要在服务器上拉取镜像，所以我们首先需要登录我们的docker hub

```docker
docker login
```

输入用户名和密码之后就好了

在构建之前，请注意你需要使用镜像的平台和你构建的平台是否是同一种架构！！！

比如你是MAC M2，那么你的ARM架构构建的镜像在AMD架构的服务器上事无法运行的，所以我们需要进行多架构构建

这是两个例子，这里创建了两个 Buildx 构建器

一个使用默认配置
另一个名为 **`mybuilder`**，并指定了支持的平台为 **`linux/amd64`** 和 **`linux/arm64`**

```docker
docker buildx create --use
docker buildx create --use --name mybuilder --platform linux/amd64,linux/arm64
```

创建完成之后，我们进行切换构建器，切换之后，我们构建的镜像默认就有**`linux/amd64`** 和 **`linux/arm64`**的了

```docker
docker buildx use mybuilder
```

**构建镜像**

```docker
docker buildx build --platform linux/amd64,linux/arm64 -t sikyyy/chat-data:latest .
```

推送容器到Docker Hub

```docker
docker buildx build --platform linux/amd64,linux/arm64 -t sikyyy/chat-data:latest --push .
```

本地加载

```docker
docker buildx build --platform linux/amd64,linux/arm64 -t sikyyy/chat-data:latest --load .
```

推送完之后，服务器那边把镜像pul下来基本上也没什么问题，然后就继续看Lan-chat这边

（阿里云真该死啊，有时候连的上github有时候连不上）

clone不成我就转思路把项目扔到gitee然后给服务器拉起来了

接下来就是老思路了，

把项目拉上来

把新的二级域名dns配好

cerbot弄个证书

nginx配个反代

go run main.go!

gogogogo

欸？怎么报错了，看看报错

```jsx
ws.js:1 Mixed Content: The page at '[https://chat.siky.online/chat](https://chat.siky.online/chat)' was loaded over HTTPS, but attempted to connect to the insecure WebSocket endpoint 'ws://chat.siky.online/ws'. This request has been blocked; this endpoint must be available over WSS.
（匿名） @ ws.js:1
Uncaught DOMException: Failed to construct 'WebSocket': An insecure WebSocket connection may not be initiated from a page loaded over HTTPS.
at [https://chat.siky.online/static/js/ws.js:1:16](https://chat.siky.online/static/js/ws.js:1:16)
Uncaught ReferenceError: socket is not defined
at sendMessage (message.js:69:9)
at handleKeyDown (message.js:147:13)
at HTMLTextAreaElement.onkeydown (chat:48:86)
```

哦哦哦，换了https应该用wss

```jsx
const socket = new WebSocket('wss://chat.siky.online/ws');
```

```go
go run main.go!
```

欸？怎么又报错了（这里忘记留报错信息了大概是说什么ssl证书什么的，报错是因为Kratos那边的nginx没配置，因为一般自己只会有一个后端服务，这次有两个，确实是忘了。而且我的kratos没有统一的前置路由，是直接大概这样来的

localhost:8000/subscribe

这样会和我的Lan-chat的后端服务冲突，我只好回Kratos给他们统一加了一层/api，改成了localhost:8000/api/subscribe，然后又是倒腾一遍镜像，然后把nginx配好了）

```go
go run main.go～
```

其实中间还有一些Fetch的报错，大家记得生产环境转线上的时候，把HTTP/HTTPS的相关请求全部要改掉哦！！！

打开网页，打开控制台，嗯，一切欣欣向荣，非常良好，正当我准备收手的时候，在线人数怎么是0

fuck

打个断点看看

![断点](https://i.miji.bid/2024/01/31/66459038859109edf39971e42ecffb2a.png)

也正常返回了啊

我去反复看了data层的Login登录和各层的调用逻辑，也没问题，也看了SADD这个封装方法也没问题

```go
// Login 登录
func (r *userRepo) Login(ctx context.Context, u *biz.User) (*redis.IntCmd, error) {
	reult := r.data.rdb.SAdd(ctx, u.Setname, u.Username)
	if reult.Err() != nil {
		return nil, reult.Err()
	}
	return reult, nil
}
```

奇了怪了

本地试一下，也是正常的。看看config.yaml，感觉也没什么问题，服务器的redis服务也启动了，让我半天想不到问题出在哪

后来检查了一下服务器的redis连接

```go
redis-cli
INFO clients
127.0.0.1:6379> INFO clients
connected_clients:1
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0
127.0.0.1:6379>CLIENT LIST
id=4 addr=127.0.0.1:51662 fd=8 name= age=2934 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=26 qbuf-free=32742 obl=0 oll=0 omem=0 events=r cmd=client
```

这里就有问题了，我连接呢，我那么大个连接去哪里了

然后我就去查我的redis连接配置，原来是我在本地，redis也没设置用户名密码什么的，就也能直接用，由于我不知道阿里云里面起的redis默认的这些是什么，我就直接用了之前注册RedisInsight拿来的免费redis服务器，好像有30MB，反正存个在线人数是一定够够的了，又不存聊天记录

这次我学聪明了，先在本地测试连接，通了之后，再转到线上

重复一次刚才的步骤！

ok

```go
cd Lan-chat
```

```go
nohup go run main.go > output.log 2>&1 & 
```

当当

![demo](https://i.miji.bid/2024/01/31/33a956f8fd0ba3e87d93be42de4f33c4.png)