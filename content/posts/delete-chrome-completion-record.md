---
title: "删除某条 Chrome 自动补全的网址记录"
date: 2016-12-21T19:53:39+08:00
draft: false
---

Chrome有一个很好又不太好的功能，就是omnibar上的自动补全网址。

经常使用Chrome的人可能会遇到这样一个情形：Chrome给补全的网址不是我们期望的，往往在下面的位置。比如下面这样：

[![](http://ww2.sinaimg.cn/large/7e921c39jw1fayd7kh394j20tx05btbu.jpg)](http://ww2.sinaimg.cn/large/7e921c39jw1fayd7kh394j20tx05btbu.jpg)
* 我提了一个PR，希望输入关键字`github`时可以自动补全到第三个，敲回车就可以直达了。
* 又或者我这个PR合入了，自动补全时不希望再出现此条目。

### [](#解决方法 "解决方法")解决方法

此时通过通过`shift + fn + delete`（Mac），`shift + alt + delete`（Windows/Linux）进行删除，操作步骤如下：

1.  输入关键字
2.  用上下键选中待删除的一条记录
3.  按上面的组合键
4.  没了

参考：
[https://productforums.google.com/d/msg/chrome/XtfQ_upcYKE/5moe0adMYC0J](https://productforums.google.com/d/msg/chrome/XtfQ_upcYKE/5moe0adMYC0J)
