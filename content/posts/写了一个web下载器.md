+++
title = '写了一个web下载器:Go-Get'
date = 2024-02-01T20:00:00+08:00
+++
这是我最早开始写的一个web项目，现在拿出来重新写一下当时的想法，也算是回顾一下

它是一个web端下载工具

前端长这样，有点丑
![前端](https://i.miji.bid/2024/02/01/0a9daf619b8e83a81f2aab3da3799819.png)

支持HTTP/HTTPS、Magenet（磁力链接）、Torrent（种子文件）的下载

相当于集成了普通的下载和BT下载（当时我是自己用BT下的比较多，就也一起集成了）

集成了Grafuna的监控面板（属于是写完下载觉得太普通了加上去的功能）

但是为什么没上线呢

没上线原因：浏览器的安全限制导致无法从前端获取本地文件夹的绝对路径，设置不了导出路径玩锤子，我哪天拿Wails把它打包一下，然后写个设置界面设置应该就好了

OK，让我们开始吧

HTTP/HTTPS部分

使用 **Query** 来获取前端传来的url

然后是尝试从传来的head中的 **Content-Disposition** 获取文件信息，文件名啊之类的

如果没有就用这个包：**"path/filepath"**

```go
fileName = filepath.Base(url)
```

拿到文件的大小，给他转成int64类型

```go
contentLength, _ := strconv.ParseInt(resp.Header.Get("Content-Length"), 10, 64)
```

开始实现分片下载

设置分片大小

按道理来讲，应该做成前端传来的值，但是我想着以后做成APP再改，自己就先凑合着用

```go
chunkSize := contentLength / 5 // 分片大小 分成5个分片，你可以根据需要更改分片数
```

外部循环

通过设置waitgroup来实现分片下载进度的体现

打算几线程下载就调整分成几个切片，管道的缓存也调整

这里用i := int64(0)是因为DownloadChunk里面类型需要，不然调的东西更多，不如只改这个

然后有多少个切片就起多少个协程

```go
var wg sync.WaitGroup
	progressCh := make(chan int, 5) // 同时下载的分片数

	// 下载分片
	for i := int64(0); i < 5; i++ {
		wg.Add(1)
		go download.DownloadChunk(url, i, chunkSize, &wg, progressCh)
	}

	// 监听分片下载进度，更新总进度条
	var totalProgress int64
	go func() {
		for p := range progressCh {
			totalProgress += int64(p)
			fmt.Printf("Total Progress: %.2f%%\n", float64(totalProgress)/float64(contentLength)*100)
		}
	}()

	// 等待所有分片下载完成
	wg.Wait()
	close(progressCh)
```

下载分片函数

```go
// DownloadChunk 下载分片的函数
func DownloadChunk(url string, chunkIndex, chunkSize int64, wg *sync.WaitGroup, progressCh chan<- int) {
	//传入URL、分片的索引、分片的大小、WaitGroup、发送分片下载进度的管道

	defer wg.Done()

	// 创建 HTTP 请求
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		fmt.Println("Error creating HTTP request:", err)
		return
	}

	// 计算分片的范围
	start := chunkIndex * chunkSize
	end := start + chunkSize - 1
	req.Header.Set("Range", fmt.Sprintf("bytes=%d-%d", start, end))

	// 发送请求并获取响应
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		fmt.Println("Error executing HTTP request:", err)
		return
	}
	defer resp.Body.Close()

	// 创建分片文件
	fileName := fmt.Sprintf("chunk_%d.tmp", chunkIndex) //分片文件名，格式为chunk_0.tmp、chunk_1.tmp等
	file, err := os.Create(fileName)                    //创建文件，返回文件指针
	if err != nil {
		fmt.Println("Error creating chunk file:", err)
		return
	}
	defer file.Close()

	// 创建进度条
	bar := progressbar.DefaultBytes(
		resp.ContentLength,
		fmt.Sprintf("Chunk %d", chunkIndex),
	)

	// 将分块下载的数据从 HTTP 响应体复制到分块文件，并通过进度条实时显示下载进度。https://github.com/schollz/progressbar配套的
	//从resp.Body复制到file，同时也复制到bar，这样就可以实时显示下载进度了
	_, err = io.Copy(io.MultiWriter(file, bar), resp.Body)
	if err != nil {
		fmt.Println("Error copying chunk:", err)
		return
	}

	// 下载完成时，发送分片下载进度给管道
	progressCh <- int(resp.ContentLength)
}
```

通过设置 HTTP 请求的 Range 头部，指定要下载的分片的字节范围

按照每个分片的索引创建分片文件并进行命名

进度条我用的是这个包https://github.com/schollz/progressbar

是按照他给的例子来写的
例子

```go

req, _ := http.NewRequest("GET", "[https://dl.google.com/go/go1.14.2.src.tar.gz](https://dl.google.com/go/go1.14.2.src.tar.gz)", nil)
resp, _ := [http.DefaultClient.Do](http://http.defaultclient.do/)(req)
defer resp.Body.Close()

f, _ := os.OpenFile("go1.14.2.src.tar.gz", os.O_CREATE|os.O_WRONLY, 0644)
defer f.Close()

bar := progressbar.DefaultBytes(
resp.ContentLength,
"downloading",
)
io.Copy(io.MultiWriter(f, bar), resp.Body)
```

我的

```go
// 创建进度条
//根据不同的切片给他们创建各自的进度条
	bar := progressbar.DefaultBytes(
		resp.ContentLength,
		fmt.Sprintf("Chunk %d", chunkIndex),
	)

// 将分块下载的数据从 HTTP 响应体复制到分块文件，并通过进度条实时显示下载进度。https://github.com/schollz/progressbar配套的
	//从resp.Body复制到file，同时也复制到bar，这样就可以实时显示下载进度了
	_, err = io.Copy(io.MultiWriter(file, bar), resp.Body)
	if err != nil {
		fmt.Println("Error copying chunk:", err)
		return
	}

	// 下载完成时，发送分片下载进度给管道
	progressCh <- int(resp.ContentLength)
```

回到外部更新总进度条

```go
// 监听分片下载进度，更新总进度条
	var totalProgress int64
	go func() {
		for p := range progressCh {
			totalProgress += p
			fmt.Printf("Total Progress: %.2f%%\n", float64(totalProgress)/float64(contentLength)*100)
		}
	}()

	// 等待所有分片下载完成
	wg.Wait()
	close(progressCh)
```

用拿到的总进度去除总文件大小，然后*100拿到百分比。

**%.2f%%将浮点数以百分比的形式打印**

最后wait一下把管道关了就好了，记得要吧分片文件合并！

来看一下合并的方法

```go
// MergeChunks 合并下载的分片
func MergeChunks(chunkCount int64, mergedFileName string) {
	// 创建合并文件，存储最终合并文件的内容
	mergedFile, err := os.Create(mergedFileName)
	if err != nil {
		fmt.Println("Error creating merged file:", err)
		return
	}
	defer mergedFile.Close()

	// 遍历所有分片文件，逐一读取内容并写入合并文件
	for i := int64(0); i < chunkCount; i++ {
		chunkFileName := fmt.Sprintf("chunk_%d.tmp", i)
		chunkFile, err := os.Open(chunkFileName)
		if err != nil {
			fmt.Println("Error opening chunk file:", err)
			return
		}
		defer chunkFile.Close()

		_, err = io.Copy(mergedFile, chunkFile)
		if err != nil {
			fmt.Println("Error copying chunk to merged file:", err)
			return
		}

		// 删除已合并的分片文件
		os.Remove(chunkFileName)
	}
}
```

因为os.Create会默认在工作目录下创建文件，所以需要指定一个path给他

我这儿传进去

```go
// 合并分片文件
	merge.MergeChunks(5, "/Users/siky/Desktop/"+fileName)
```

接下来就是按照之前设置的名字格式，按顺序打开文件，写入到创建好的文件里面，然后把分片删了就好了

这就是一个http/https分段下载并合并文件的过程了

Magenet（磁力链接）

首先我们需要知道Magenet和Torrent的关系

**Torrent文件：**

- 是一个包含了关于要下载文件的元数据的文件，通常以 **`.torrent`** 为扩展名。
- 包含了文件的结构、大小、哈希值等信息，以及用于连接到Tracker服务器的信息。
- 用户通常通过下载Torrent文件来开始BitTorrent下载。

**Magnet链接：**

- 是一种基于URI（Uniform Resource Identifier）的链接，可以用来标识和定位资源，例如BitTorrent中的文件。
- 包含了与Torrent文件相似的信息，如文件名、大小、哈希值等。
- 不需要事先下载一个独立的Torrent文件，可以直接使用Magnet链接开始下载。

我用的是这个包[https://github.com/anacrolix/torrent](https://github.com/anacrolix/torrent)

```go
func MagnetDownload(c *gin.Context) {
	// 磁力链接
	magnetURL := c.Query("magnetURL")
	//url := "magnet:?
	// 传入磁力链接和下载目录
	outputCh := make(chan string, 10000)
	go func() {
		defer close(outputCh)
		download.DownloadMagnetFile(magnetURL, "/Users/siky/go/src/Go-Get", outputCh)
		c.JSON(http.StatusOK, gin.H{"message": "Download completed"})
		// 删除 .torrent.db 文件
		dbFilePath := filepath.Join("/Users/siky/go/src/Go-Get", ".torrent.db")
		err := os.Remove(dbFilePath)
		if err != nil {
			way.SendOutput(outputCh, "删除 .torrent.db 文件时出错:%v", err)
		} else {
			way.SendOutput(outputCh, ".torrent.db 文件已成功删除")
		}
		way.SendOutput(outputCh, "------------------------该文件下载完成------------------------")
	}()
	//处理输出消息并发送到 WebSocket 客户端
	go func() {
		for message := range outputCh {
			for client := range connectedClients {
				if err := client.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
					fmt.Println("Failed to send data to client:", err)
				}
			}
		}
	}()
}
```

这个管道的缓存当时随便定的，因为这个管道是拿来给前端发消息的，本地测试嘛，反正也无所谓

```go
// SendOutput 将输出发送到前端
func SendOutput(outputCh chan<- string, format string, args ...interface{}) {
	output := fmt.Sprintf(format, args...)
	fmt.Println(output) // 可选，用于在服务器端打印输出
	outputCh <- output
}
```

DownloadMagnetFile方法

```go
// DownloadMagnetFile 添加一个磁力链接并启动下载任务
func DownloadMagnetFile(magnetURL, downloadDir string, outputCh chan<- string) {
	// 创建一个新的客户端
	client, err := torrent.NewClient(nil)
	if err != nil {
		log.Fatal(err)
	}
	way.SendOutput(outputCh, "客户端初始化成功")
	defer client.Close()

	// 解析磁力链接，返回一个 MagnetFile 的指针
	magnetFile, err := client.AddMagnet(magnetURL)
	if err != nil {
		way.SendOutput(outputCh, "解析磁力链接失败:%v", err)
		log.Fatal(err)
	}
	way.SendOutput(outputCh, "解析磁力链接成功")

	// 等待解析磁力链接，当元数据下载完成后，该通道会被关闭，此时可以访问种子信息
	// 如果超时，就会执行 time.After() 中的代码
	select {
	case <-magnetFile.GotInfo():
		way.SendOutput(outputCh, "开始获取元信息")
	case <-time.After(15 * time.Second):
		way.SendOutput(outputCh, "获取元信息，建议检查一下网络情况，或是磁力链接是否失效")
		return // 或者执行其他超时后的操作
	}

	// way.SendOutput(outputCh, "开始获取元信息")
	// <-torrentFile.GotInfo()

	info := magnetFile.Info()

	way.SendOutput(outputCh, "总文件名称:%v", info.Name)
	way.SendOutput(outputCh, "总文件数量:%v", info.TotalLength()+1)
	way.SendOutput(outputCh, "总文件大小:%v", len(info.Files))

	files := magnetFile.Files()
	for _, file := range files {
		way.SendOutput(outputCh, "文件名称:%v", file.Path())
		way.SendOutput(outputCh, "文件大小:%v", file.Length())
	}

	// 获取种子文件的原始文件名
	magnetFileName := filepath.Base(magnetFile.Info().Name)
	way.SendOutput(outputCh, "文件原始名称:%v", magnetFileName)
	// 获取种子文件的原始文件名（不包含扩展名）
	magnetFileNameWithoutExtension := strings.TrimSuffix(magnetFileName, filepath.Ext(magnetFileName))
	//使用正则表达式匹配动画名称
	animeName := getname.ExtractAnimeName(magnetFileNameWithoutExtension)

	// 设置保存目录
	savePath := filepath.Join(downloadDir, animeName)
	way.SendOutput(outputCh, "保存目录:%v", savePath)

	// 创建目录（如果不存在），默认权限是0777
	if err := os.MkdirAll(savePath, os.ModePerm); err != nil {
		log.Fatal(err)
		way.SendOutput(outputCh, "创建目录失败:%v", err)
	} else {
		way.SendOutput(outputCh, "创建目录成功")
	}

	// 设置下载目录，如果是部分下载，就可以指定保存位置，全部下载保存位置就是默认的下载目录
	downloadDir = savePath

	// 更改当前工作目录到下载目录
	if err := os.Chdir(savePath); err != nil {
		log.Fatal(err)
	}

	magnetFile.AllowDataDownload() // 允许下载数据
	way.SendOutput(outputCh, "允许下载数据")

	// 下载所有文件
	magnetFile.DownloadAll()
	way.SendOutput(outputCh, "开始下载")

	// 等待下载完成
	client.WaitAll()
	way.SendOutput(outputCh, "下载完成:"+magnetFileName)
}
```

基本上就是这个样子，顺便用正则来匹配了一下下载的文件名，然后创建对应的文件夹，把下载的对应文件扔在里面，由于我测试的都是番剧的磁链或种子，他们的名字有些比较抽象，导致正则不是很好搞，有些能配上有些不行…

Torrent

下载和解析基本和Magenet一样这里就不重新写的

不过Torrent文件会被我上传到Mongodb

Mongodb在我们使用驱动程序进行数据交互的时候，通常将数据以BSON格式进行编码和解码

```go
func UploadToMongoDB(c *gin.Context) {
	// 创建一个 downloadInfo 变量，用于存储 JSON 数据
	var downloadInfo data.UploadTorrentInfo

	// 从请求中读取 JSON 数据并解码到 downloadInfo 变量
	if err := c.BindJSON(&downloadInfo); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	//把json转化为BSON
	uploadinfo := bson.M{
		"name":          downloadInfo.Name,
		"files_num":     downloadInfo.FilesNum,
		"total_length":  downloadInfo.TotalLength,
		"info_hash":     downloadInfo.InfoHash,
		"info_bytes":    downloadInfo.InfoBytes,
		"announce":      downloadInfo.Announce,
		"comment":       downloadInfo.Comment,
		"created_by":    downloadInfo.CreatedBy,
		"creation_date": downloadInfo.CreationDate,
		"upload_time":   downloadInfo.UploadTime,
	}

	// 在此处将下载信息插入 MongoDB 数据库
	data.InsertDocument(dbinit.Client, uploadinfo, "Go-Get-MongoDB", "Torrents")

	c.JSON(http.StatusOK, gin.H{"message": "Data inserted successfully"})
}
```

关于监控

这部分我会在下一篇进行仔细说明，这里先粗略写

使用了Grafuna进行数据指标的监控

使用Prometheus作为数据源

使用pushgateway作为数据上报

这里面有我定义的一些常量

```go
const (
	PrometheusUrl          = "http://127.0.0.1:9091" //Prometheus PushGateway 的地址，用于推送指标数据。
	PrometheusJob          = "go_get_prometheus_qps" //Prometheus job 名称，用于标识这个应用程序的任务。,需要和yaml的job_name一致
	PrometheusNamespace    = "go_get_data"           //Prometheus 指标的命名空间。
	EndpointsDataSubsystem = "endpoints"             //Prometheus 指标的子系统，用于更细粒度地标识指标
)

var (
	Pusher *push.Pusher //Prometheus pusher，用于将指标数据推送到 Prometheus PushGateway。

	//一个 Counter 向量，用于记录接口 QPS 的统计信息.用于记录每个接口的请求数量。
	EndpointsQPSMonitor = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: PrometheusNamespace,    //命名空间
			Subsystem: EndpointsDataSubsystem, //子系统
			Name:      "QPS_statistic",        //指标名称
			Help:      "统计QPS数据",              //帮助信息
		}, []string{EndpointsDataSubsystem}, //用于标识不同的接口
	)
	//一个 Histogram 向量，用于记录接口耗时的统计信息。用于记录每个接口的耗时。
	EndpointsLantencyMonitor = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: PrometheusNamespace,                                      //命名空间
			Subsystem: EndpointsDataSubsystem,                                   //子系统
			Name:      "lantency_statistic",                                     //指标名称
			Help:      "统计耗时数据",                                                 //帮助信息
			Buckets:   []float64{1, 5, 10, 20, 50, 100, 500, 1000, 5000, 10000}, //耗时区间，如果一个请求的耗时在某个区间内，就会被记录到对应的桶中。
		}, []string{EndpointsDataSubsystem}, //用于标识不同的接口
	)
)
```

当然了，记得Init一下

```go
// 初始化指标
func Init() {
	Pusher = push.New(PrometheusUrl, PrometheusJob)
	prometheus.MustRegister(
		EndpointsQPSMonitor,
		EndpointsLantencyMonitor,
	)
	Pusher.Collector(EndpointsQPSMonitor)
	Pusher.Collector(EndpointsLantencyMonitor)
}
```

基于这些构建了这两个中间件

```go
// 用于在接口被调用时递增相应的 QPS 计数器，并将 endpoint 作为 label。
func HandleEndpointQps() gin.HandlerFunc {
	return func(c *gin.Context) {
		//获取当前请求的接口路径
		endpoint := c.Request.URL.Path
		fmt.Println(endpoint)
		//递增 QPS 计数器，将当前请求的 endpoint 作为 label。然后，通过 Inc 方法递增 QPS 计数器。
		// 排除 /metrics 路由
		// if endpoint != "/metrics" {
		metrics.EndpointsQPSMonitor.With(prometheus.Labels{metrics.EndpointsDataSubsystem: endpoint}).Inc()
		// }
		c.Next()
	}
}

func HandleEndpointLantency() gin.HandlerFunc {
	return func(c *gin.Context) {
		// 获取当前请求的接口路径
		endpoint := c.Request.URL.Path
		// 记录当前时间，作为请求处理开始的时间点
		start := time.Now()
		defer func(c *gin.Context) {
			// 计算请求处理耗时
			lantency := time.Since(start)
			// 将耗时直接转换为毫秒，保留三位小数
			lantencyFloat64 := float64(lantency.Nanoseconds()) / 1e6

			fmt.Printf("请求接口:%v，请求处理耗时：%f ms\n", endpoint, lantencyFloat64)
			// 将当前请求的 endpoint 作为 label。然后通过 Observe 方法记录耗时。
			metrics.EndpointsLantencyMonitor.With(prometheus.Labels{metrics.EndpointsDataSubsystem: endpoint}).Observe(lantencyFloat64)
		}(c)
		c.Next()
	}
}
```

让他们在所有的路由中调用

```go
r := gin.Default()

// 设置qps中间件,用于统计接口qps//设置接口耗时中间件，用于统计接口耗时
r.Use(middleware.HandleEndpointQps(), middleware.HandleEndpointLantency())
```

```go
/metrics
请求接口:/metrics，请求处理耗时：1.300958 ms
[GIN] 2024/02/01 - 19:37:01 | 200 |    1.507875ms |       127.0.0.1 | GET      "/metrics"
/
请求接口:/，请求处理耗时：1.181958 ms
[GIN] 2024/02/01 - 19:37:04 | 200 |     1.28625ms |             ::1 | GET      "/"
```

就像这样

![例子](https://i.miji.bid/2024/02/01/b34219b5687e42904a7c245ca3831768.png)