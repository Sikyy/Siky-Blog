+++
title = '关于Mac终端lsof无法显示正常进程名字'
date = 2024-02-05T01:00:00+08:00
+++


因为项目里需要获取进程名字，就有了这篇Blog

当我在获取我的443端口上监听的进程的时候，我发现了一些奇怪的进程，他们的名字要么显示不全，要么无法显示

```bash
siky at Siky_Macmini in ~ 
$ lsof -i:443
COMMAND     PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
Code\x20H 94659 siky   22u  IPv4 0xcdf758597a6c83c3      0t0  TCP siky-mac-mini:53734->20.205.69.80:https (ESTABLISHED)
?\x93\x94 97161 siky   59u  IPv4 0xcdf758597a75ca6b      0t0  TCP siky-mac-mini:53791->111.170.130.241:https (ESTABLISHED)
```

在我使用ps aux | grep 进行排除后发现进程原本的名字应该是
Code Helper

哔哩哔哩 Helper

```bash
siky at Siky_Macmini in ~ 
$ ps aux | grep 97161
siky             **97161**   0.3  0.4 409412752  30192   ??  S     1:11AM   0:00.94 /Applications/哔哩哔哩.app/Contents/Frameworks/哔哩哔哩 Helper.app/Contents/MacOS/哔哩哔哩 Helper --type=utility --utility-sub-type=network.mojom.NetworkService --lang=zh-CN --service-sandbox-type=network --no-sandbox --host-rules=MAP bilipc.bilibili.com localhost:53536 --user-data-dir=/Users/siky/Library/Application Support/bilibili --shared-files --field-trial-handle=1718379636,r,9439795961709248486,3376730300454549434,131072 --enable-features=PlatformHEVCDecoderSupport --disable-features=SpareRendererForSitePerProcess
siky             97846   0.0  0.0 408652496   1616 s000  S+    1:14AM   0:00.00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox **97161
$ ps aux | grep 94659
siky             94659   0.0  0.2 441901040  15200   ??  S    12:24AM   0:01.00 /Applications/Visual Studio Code.app/Contents/Frameworks/Code Helper.app/Contents/MacOS/Code Helper --type=utility --utility-sub-type=network.mojom.NetworkService --lang=zh-CN --service-sandbox-type=network --user-data-dir=/Users/siky/Library/Application Support/Code --standard-schemes=vscode-webview,vscode-file --enable-sandbox --secure-schemes=vscode-webview,vscode-file --cors-schemes=vscode-webview,vscode-file --fetch-schemes=vscode-webview,vscode-file --service-worker-schemes=vscode-webview --code-cache-schemes=vscode-webview,vscode-file --shared-files --field-trial-handle=1718379636,r,5154262016076523702,16136674051785448622,262144 --disable-features=CalculateNativeWinOcclusion,SpareRendererForSitePerProcess --seatbelt-client=40
siky             97787   0.0  0.0 408626896   1392 s000  S+    1:14AM   0:00.00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox 94659**
```

我的第一反应是检查我的字符集

后来又突然想到Code Helper也不是中文啊，估计是中间这个空格也有问题

估计是lsof在进行分段识别的时候把这个空格也包含进去了，搞出了问题，那为什么哔哩哔哩无法显示呢，我检查了我终端的语言环境和文本编码：UTF-8也没问题啊，而且后面的ps aux | grep 97161中也能正常显示中文，估计是lsof -i在识别COMMAND的时候有些问题

那我我现在的项目需要拿到正确的COMMAND该怎么做呢

```bash
lsof -i:443 -F
```

大致就能拿到这样的数据，但是如果多个端口监听的话，那么就会继续循环下去

```bash
//数据是这样的
p1190
g644
R644
cGoogle Chrome Helper
u501
Lsiky
f29
au
l
tIPv4
G0x10007;0x0
d0xcdf758597a6d43c3
o0t0
PTCP
nsiky-mac-mini:61472->221.229.52.145:https
TST=ESTABLISHED
TQR=0
TQS=0

```

他们都有一个前标识符号，方便我们通过这些标识符号去进行对应的解析

这时你就可以根据你拿到的数据进行你需要的解析啦

不过我现在还是没有找到让我在终端正常显示中文进程名字的方法，也许是lsof有提供参数，但是我没太找到，最近有很多事情要忙，先放一下

而且COMMAND 对应的名字虽然它转成了十六进制编码，但是也显示不全，估计是也做了字符的限制，我感觉还是去看看源码好一点