---
title: "下载老版本 iOS SDK 开发文档"
date: 2016-12-21T19:52:12+08:00
draft: false
---

众所周知，苹果是一个“极度激进”的公司，发布iOS10，就想让你放弃iOS9。可国内的现状是想摆脱iOS8都困难，不支持iOS7都会是一个很艰难的决定。同时，升级Xcode会把旧版本的开发文档移除，开发者只能乖乖地下载新版文档。

不知道是不是我的需求有些奇葩，我竟然想要找旧版本的iOS开发文档，如iOS8。不过难免会有这方面的需求，遇到问题毫不退缩迎难而上，是RD的基本素养。（喂，自夸也要有个限度啊）

### 工具准备

* Dash
    Dash是Mac上非常优秀的API文档查看软件，集成了150+种语言和框架的开发文档，主打即时搜索离线文档，效率利器。售价$24.99，略贵，但也很公道。
    
    前段时间苹果把Dash各平台的app都做了下架处理，所以如果需要购买可以去官网。对此事件稍微有些了解，双方各执一词，不多作评价。但这丝毫不影响Dash是一款非常棒的应用。
    
    需要Dash主要是因为我们可以把下载下来的旧版本docset放到目标位置，让dash可以扫描到并加载，我们使用Dash进行文档的查看，Xcode已经指望不上了。
    
* BetterZip
    
    有个功能类似于WinRAR，可以在解压之前查看压缩的内部文件列表。嗯，仅此而已。没有也可以。不过我不喜欢把几百M的压缩包解压完再看对比里面的东西是不是如预期。
    

### 探索

Google了一段时间，换了各种关键字，终于从[Downloading Old iOS SDK Documentation](http://useyourloaf.com/blog/downloading-old-ios-sdk-documentation/)的评论中找到一些线索。该篇博文由于时代久远信息陈旧放到Xcode8.0的现在已不适用，不过评论还是有新鲜的。

> [![](http://ww1.sinaimg.cn/large/7e921c39gw1fayun5r4twj210y0i441o.jpg)](http://ww1.sinaimg.cn/large/7e921c39gw1fayun5r4twj210y0i441o.jpg)

Big Will给出的两个链接均是devimages.apple.com域的，并依然可用，下载下来是dmg格式的。于是发散思维，转而去寻找dmg格式历史文档了。

上面给出的链接为：[http://devimages.apple.com/docsets/20131022/031-1317-A.dmg](http://devimages.apple.com/docsets/20131022/031-1317-A.dmg)

搜索关键字：devimages.apple.com，docset，2014

因为我要找iOS8的文档。Google结果第一条：
[https://developer.apple.com/library/downloads/docset-index.dvtdownloadableindex](https://developer.apple.com/library/downloads/docset-index.dvtdownloadableindex)

就是它了，是一个xml格式的文件。截取iOS8相关部分：
```xml
<!-- START iOS 8 -->
<dict>
    <key>fileSize</key>
    <integer>438780988</integer>
    <key>identifier</key>
    <string>com.apple.adc.documentation.AppleiOS8.0.iOSLibrary</string>
    <key>name</key>
    <string>iOS 8</string>
    <key>source</key>
    <string>https://devimages.apple.com.edgekey.net/docsets/20140917/031-06020-A.dmg</string>
    <key>userInfo</key>
</dict>
```

倒数第二行的`<string>`标签内的链接就是我们要找的dmg API文档了。

加载dmg文件，使用BetterZip查看里面的文件，是pkg。

不过凭程序员的直觉，这个pkg也不过是个zip文件，改后缀果然能解压。 继续改后缀，解压，改后缀，解压，然后就得到了我们熟悉的`.docset`文档。

简易流程图：（懒癌晚期，就不分别截图了）
```
031-06020-A.dmg -> 
iOSDocset.pkg   --rename-->  iOSDocset.zip  -> 
Payload 		--rename-->  Payload.zip 	-> 
Payload 		--rename-->  Payload.zip 	-> # 没错，包了两层
com.apple.adc.documentation.AppleiOS8.0.iOSLibrary.docset
```

接下来把docset文件放入
`~/Library/Developer/Shared/Documentation/DocSets/`

进入Dash的设置Docsets，点rescan就会搜索到新加入的API文档。然后Do whatever you want。

### 备份

下面对 docset-index.dvtdownloadableindex 做一个备份：
[docset-index.dvtdownloadableindex](/Download-old-version-iOS-SDK-documentation/docset-index.dvtdownloadableindex "docset-index.dvtdownloadableindex")
