+++
title = '年轻人的第一个App'
date = 2024-01-29T01:30:00+08:00
+++


基于Wails构建的年轻人第一个App：Siky-Net

GitHub：https://github.com/Sikyy/wails-NetPackage

今天把Siky-Net打包成dmg了

第一次打包dmg文件，学习了一下，原来是是要用磁盘工具先分一个dmg空间

这会生成一个xx.dmg 和 xx 的硬盘空间

来把需要的app文件和Applications的软链接传到创建的xx的硬盘空间里

如果有需要的话就再塞一个背景和icns图标

最后把图标调整为适合的样子，然后把创建的硬盘空间弹出

打开磁盘工具点击菜单栏 映像=>转换，选择刚才dmg文件，命名后点击转换就好了

我临时凑了几张图片长这个样子

![Siky-Net-DMG.png](https://i.miji.bid/2024/01/29/7a9f1bfaa29c76868ec0a76de293e739.png)

对Wails框架的使用体验：

当前版本v2.7.0

打包大小很小，上手很快

简体中文文档支持不错

对Golang非常友好，支持Go方法在前端直接调用

不足点：

内置的Event事件写的很模糊，没有给例子，害得我边试边找相关例子，还好最后成功调用了

打算到时候实现一个最小复现实例看看Wails能不能给我水个PR

内置的Event事件官方的定义是这样的：

Wails 运行时提供了一个统一的事件系统，其中事件可以由 Go 或 JavaScript 发出或接收。 可选地，数据可以与事件一起传递。 侦听器将接收本地数据类型中的数据。

相当于一个很方便的数据调用工具（超级超级方便）

这里会需要先引入"github.com/wailsapp/wails/v2/pkg/runtime"包

```go
"github.com/wailsapp/wails/v2/pkg/runtime"
```

以下是我的例子：

```go
// 进行数据包的捕获
func (a *App) CaptureTraffic() {
	SessionInfoCh = make(chan define.SessionInfoFront)

	go services.CaptureTraffic(SessionInfoCh)

	go func() {
		for tabelinfo := range SessionInfoCh {
			// 使用 EventsEmit 方法触发事件并传递 tabelinfo 数据
			runtime.EventsEmit(a.ctx, "captureTraffic", tabelinfo)
		}
	}()

}
```

调用runtime.EventsEmit

这是这个方法的官方解释

### EventsEmit 触发指定事件[](https://wails.io/zh-Hans/docs/next/reference/runtime/events#eventsemit--%E8%A7%A6%E5%8F%91%E6%8C%87%E5%AE%9A%E4%BA%8B%E4%BB%B6)

此方法触发指定的事件。 可选数据可以与事件一起传递。 这将触发任意事件侦听器。

```go
Go: `EventsEmit(ctx context.Context, eventName string, optionalData ...interface{})`

JS: `EventsEmit(eventName: string, ...optionalData: any)`

runtime.EventsEmit(a.ctx, "captureTraffic", tabelinfo)
```

传入一个ctx，“事件名称”，任何类型的数据

这时需要来到前端接收

这里我由于是需要持续监听所以我选择了EventOn方法

### EventsOn 添加事件侦听器[](https://wails.io/zh-Hans/docs/next/reference/runtime/events#eventson--%E6%B7%BB%E5%8A%A0%E4%BA%8B%E4%BB%B6%E4%BE%A6%E5%90%AC%E5%99%A8)

此方法为给定的事件名称设置一个侦听器。 当 [触发指定事件](https://wails.io/zh-Hans/docs/next/reference/runtime/events#%E8%A7%A6%E5%8F%91%E6%8C%87%E5%AE%9A%E4%BA%8B%E4%BB%B6) 名为 `eventName` 类型的事件时，将触发回调。 与触发事件一起发送的任何其他数据都将传递给回调。 它返回 一个函数来取消侦听器。

```go
Go: `EventsOn(ctx context.Context, eventName string, callback func(optionalData ...interface{})) func()`

JS: `EventsOn(eventName string, callback function(optionalData?: any)): () => void`
```

在调用之前需要import进来

```jsx
import* { EventsOn } *from* '../../wailsjs/runtime/runtime';
```

```jsx
function GetTraffic() {
  EventsOn('captureTraffic', (data) => {
    console.log("接收到流量数据:", data);

    // 检查数据的ID是否已经存在于tableData中
    const existingIndex = tableData.value.findIndex(item => item.id === data.ID);

    if (existingIndex !== -1) {
      // 如果存在，更新现有数据
      tableData.value[existingIndex] = {
        id: data.ID,
        date: data.Time_s,
        clientname: "",
        status: data.Status,
        upload: data.SessionUpTraffic,
        download: data.SessionDownTraffic,
        still_time: data.Length_of_time + "ms",
        method: data.Method,
        host: data.Host,
      };
    } else {
      // 如果不存在，添加新数据
      tableData.value.push({
        id: data.ID,
        date: data.Time_s,
        clientname: "",
        status: data.Status,
        upload: data.SessionUpTraffic,
        download: data.SessionDownTraffic,
        still_time: data.Length_of_time + "ms",
        method: data.Method,
        host: data.Host,
      });
    }
  });
}
```

关于Siky-Net

这个项目最早是我刚学完go没多久，在写完一个web端下载工具的时候想写的，当时感觉用Gin写完web项目之后感觉还是太水了，想要实现一个复杂一点了微服务项目，然后觉得Go的网络和并发性能比较好就写了这个请求查看器，后来发现这个功能写微服务都写不了几个文件，除非加上代理之类的还有前端页面，于是就把网络部分写了就搁置了。原项目地址：https://github.com/Sikyy/view-net

这个项目还是因为ElEment UI救了它一命，不然我估计也不会碰它

这次算是我Wails初试水

在我搞懂Wails的Event之后感觉想写一万个App，写App的感觉实在是太好了

感觉Blog也要勤快更新了

Siky

2024.1.29 01:24