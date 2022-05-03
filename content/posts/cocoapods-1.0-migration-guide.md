---
title: "CocoaPods 1.0 迁移指南"
date: 2016-08-05T19:43:28+08:00
draft: false
---

### TL;DR

- 必须定义`targets`，之前未定义的话会默认使用第一个target
- 定义的`target`必须表示一个Xcode target
- 如果需要表示用在多个target中一系列pods的集合，可以使用`abstract_pods`，然后在里面添加target和单独的pods。嵌套的target继承外部的pods依赖
- 像Tests targets这样，需要知道一个target的pods但又不需要链接(link)的target，可能通过他们的搜索路径来定义继承自外部pods (`inherit! :search_paths`)
- 因为上面的定义，`exclusive => true`和`link_with`被移除
- `xcodeproj`被重命名为`project`使 [DSL](https://en.wikipedia.org/wiki/Domain-specific_language)[^1] 词汇保持一致
- `pod`的`:head`被移除，使用`:podspect`或者通过`git`, `:svn`等直接指定远程repo
- 可以在`Podfile`里通过`install!`直接指定安装选项 （比如，`install! 'cocoapods', :deterministic_uuids => false, :integrate_targets => false`)
- 之前被标记为废弃的DSL属性都被移除了
- 引入`pod trunk delete`和`pod trunk deprecate`
- `pod search`被优化，现在可以搜索私有spec repos了
- `pod lib lint`和`pod spec lint`会验证pod是否会被实际引入

### Podfile 变化

#### 必须指明`target`

之前如果只有一个`target`，不写也不会有问题，会默认使用那个`target`。
```objc
# 之前
platform :ios, '7.0'
inhibit_all_warnings!

pod 'SDWebImage','~> 3.7.1'
pod 'SDWebImage/WebP'
pod 'libextobjc', '~> 0.4'
```

```objc
# 之后
platform :ios, '7.0'
inhibit_all_warnings!

target :mainApp do
	pod 'SDWebImage','~> 3.7.1'
	pod 'SDWebImage/WebP'
	pod 'libextobjc', '~> 0.4'
end
```

#### 新增`abstract_target`

在1.0之前，如果有多个target要使用相同的一些pod，大概有3种方法，从low到不low排序是：

1.  **每个target都写一遍**
```objc
platform :ios, '7.0'
inhibit_all_warnings!

target :mainApp do
	pod 'SDWebImage','~> 3.7.1'
	pod 'SDWebImage/WebP'
	pod 'libextobjc', '~> 0.4'
end

target :subApp do
	pod 'AFNetworking'
	pod 'libextobjc', '~> 0.4'
end
```

2.  **`link_with`**
```objc
link_with 'mainApp', 'subApp'

platform :ios, '7.0'
inhibit_all_warnings!

pod 'libextobjc', '~> 0.4'

target :mainApp do
	pod 'SDWebImage','~> 3.7.1'
	pod 'SDWebImage/WebP'
end

target :subApp do
	pod 'AFNetworking'
end
```

找遍网上已经找不到完整的示例，这个是我猜测的大概用法。没有亲自尝试，因为实际场景应该不使用这种方式。

3.  **ruby函数def的方式**
```objc
platform :ios, '7.0'
inhibit_all_warnings!

def shared_pods
	pod 'libextobjc', '~> 0.4'
end

target :mainApp do
	shared_pods
	pod 'SDWebImage','~> 3.7.1'
	pod 'SDWebImage/WebP'
end

target :subApp do
	shared_pods
	pod 'AFNetworking'
end
``` 
可以说这种解决方式在1.0之前还算是很优雅的，灵活性很高。


**使用`abstract_pods`的方式**
```objc
# 注意: Xcode项目中没有一个叫"Shows"的target

abstract_target 'Shows' do
  pod 'ShowsKit'
  pod 'Fabric'

  # ShowsKit + ShowWebAuth
  target 'ShowsiOS' do
    pod 'ShowWebAuth'
  end

  # ShowsKit + ShowTVAuth
  target 'ShowsTV' do
    pod 'ShowTVAuth'
  end
end
```

#### `exclusive => ture`和`link_with`被移除

由于`abstract_target`的引入，这两个属性的作用就不大了。完全可以使用`abstract_target`代替。

在1.0处于beta阶段时，这两个属性就被标记为废弃，在1.0正式版的时候移除。 如果不了解这两个属性也没有关系，之前没有使用过后面也不会使用了。

#### `pod`的`:head`被移除

由于会被人误解其本意而被移除，详见[这个issue。](https://github.com/CocoaPods/CocoaPods/issues/4673)

#### `:local`被移除，使用`:path`代替

#### `xcodeproj`被重命名为`project`

### CocoaPods 变化

#### 完整cloneSepcs仓库，而非之前的shallow clone

shadllow clone使用`git clone`加上`--depth`参数指定只clone特定数目的历史版本。由于此种方式会给git服务器带来额外的压力，所以有此点变化。详情见[这个issue](https://github.com/CocoaPods/CocoaPods/issues/5016)。

这样会变得异常慢，我司有CocoaPods官方Specs镜像，所以建议直接使用以加速。

#### `pod install`不再先更新repo

在1.0之前，`pod install`命令会默认先执行`pod repo update`导致每次`pod install`会消耗较多时间，想改变以行为需要加上`--no-repo-update`选项。其实大部分的时候我们并不此默认行为，以至于我直接创建在`.zshrc`里面创建了一个`alias pin="pod install --no-repo-update"`来简化操作。
现在没有默认的更新行为了，`pod install`就是直接安装。报错了或者要更新特定版本时再运行`pod repo update`吧。
或者你喜欢原来的行为？那么`pod install --repo-update`

#### Development Pods默认不再是unlock状态

如果你更改了你的开发中的依赖的约束，并且想覆盖本地锁定的版本，可以手动执行 `pod update ${DEPENDENCY_NAME}`

### Podspec 变化

#### `platform`和`deployment_target`

原来的podspec文件中如果有`s.ios.platform`, `s.osx.platform`类似的语法，那么在1.0以后的版本会报错：

```
[!] Invalid `WebViewJavascriptBridge.podspec` file: undefined method `platform=’ for #  
Did you mean? platform.
```

这里podspec的变化是移除了`.ios.`和`.osx.`，支持单平台直接使用`s.platform = :ios, '7.0'`的方式，如果支持平台使用新增的`s.ios.deployment_target = '7.0'`, `s.osx.deployment_target = '10.10'`的方式。当然也可以使用`s.platforms = { :ios => '7.0', :osx => '10.10' }`。


[^1]: 如果你写过 makefile 或者使用 CSS 设计网页，那么你已经用过 DSL 了，可以翻译成**特定域语言**。 DSL 是设计的一种极小又富有表现力的**编程**语言来完成特定任务。
