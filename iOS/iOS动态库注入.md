# iOS动态库注入
# 前言
前段时间，我们遇到了一个小麻烦。

我们部门的工作是提供一些底层的SDK，以`.framework`的形式交付，但是测试的时候仅靠demo没法复现一些问题，所以有时需要直接使用最终的成品app来测试。跟我们合作的项目组不可能将他们的源码提供出来，这就意味着我们得一遍一遍地交付，然后一遍一遍地测试，整个流程非常繁琐。

后来，通过查阅一些资料，了解到一种动态库注入的方案，非常符合我们的场景，便学以致用，最终效果非常好。

本文会将动态库注入的技术细节梳理出来，并且对网络中的一些不详实的地方加以补充和注释，以便感兴趣的同学少走弯路。
<blockquote>
注：

- 本文仅讨论一些动态库注入的基础知识和技巧，不涉及具体注入对象

- 请勿对市场上的应用进行此类操作，否则后果自负

- 转载请注明出处
</blockquote>>

# 什么是动态库注入？
我们都知道，共享程序代码有两种常用的方式，一种是静态库，一种是动态库。
不同于静态库在编译时链接到主程序的可执行文件中，动态库是一种独立的程序包，一般是在装载（`load`）或运行时（`runtime`）加载。
而动态库注入，就是在加载之前，将原有的动态库替换掉，使得加载的是我们需要的动态库。这样，我们就可以按照我们自己的需求修改动态库功能，而不需要重新编译可执行文件。
本文讨论的均为静态注入，也就是在app未运行时进行注入操作。

# 为什么要注入动态库？
我能想到的一些应用场景，包括但不限于：
- 测试调试
- bug修复
- 动态加载/热修复(由于[审核政策](https://developer.apple.com/app-store/review/guidelines/#hardware-compatibility)限制，`iOS App Store`正式包不能使用)

# 如何注入动态库？
那么，既然动态库注入有这么多用处，我们该如何操作呢？下面我们就一一展开。
## 准备工作
首先，我们需要准备好一些资源。
### 被注入的iPa包
通常的逆向手段，使用的都是正式包，正式包涉及到砸壳。至于怎么砸壳，不在本文讨论范围内，感兴趣的同学可以自行查找。

想知道一个`.ipa`是否需要砸壳，可以通过`otool`命令来查看：
```shell
otool -l NeedReCodeSign | grep cryp
```
这里的`NeedReCodeSign`是`.ipa`文件解压后的的`.app`包内容里的二进制文件，查询结果中`cryptid`为`0`代表无需砸壳。

因为我们与项目组有合作关系，所以可以更方便拿到开发包（使用development方式导出），来进行注入。开发包是无壳状态，我们可以省去砸壳的步骤，更加方便。

我们先准备一个`.ipa`文件，且称为`NeedReCodeSign.ipa`。
### 待注入的动态库文件
可以是`.framework`或`.dylib`，编译时签不签名都没有关系。
### 签名证书
苹果开发者证书，可以通过在终端输入以下命令来查询：
```shell
security find-identity -v -p codesigning
```
选择合适的证书（所谓合适的证书，是指跟`.ipa`文件的`bundle id`一致的证书，当然如果没有与`.ipa`文件的`bundle id`一致的证书也没关系，后面的“重命名BundleId”和“重签名app”会介绍到）。

本文以`Apple Development: *** (*********)`为例。
### 描述文件
可以使用原有项目，也可以新建一个项目，使用签名证书和描述文件打包，然后取出产物“.app”目录下的"embedded.mobileprovision"文件。
### 导出新的entitlements
#### 半自动导出
准备好的`embedded.mobileprovision`文件，使用如下命令查看`entitlements`信息：
```shell
security cms -D -I embedded.mobileprovision > provision.plist
```
找到`provision.plist`的`Entitlements`key，然后将其内容拷贝出来，存放到一个空的文件中，保存为`entitlements_ori.plist`。
#### 命令导出
使用管道命令来生成（需安装`PlistBuddy`）：
```shell
/usr/libexec/PlistBuddy -x -c 'Print:'Entitlements'' provision.plist > entitlements_ori.plist
```
## 执行过程
### 解压`.ipa`文件
跟获取描述文件一样，我们先将`NeedReCodeSign.ipa`文件的后缀名改为`.zip`，解压后得到“Payload”文件夹。
### 替换动态库文件
右键点击`Payload/NeedReCodeSign.app`，选择`Show Package Contents`进入包内容目录，打开`Frameworks`文件夹，使用新的动态库文件替换掉原有的文件。

注：`.framework`实际上是一个文件夹，需要替换而不是合并。
### 重签名`Frameworks`
使用命令，将`Frameworks`文件夹中的所有文件进行重签名：
```shell
codesign -fs "Apple Development: *** (*********)" xxx.framework
# or
codesign -fs "Apple Development: *** (*********)" xxx.dylib
```
### 替换描述文件
在`Frameworks`同级目录，找到`embedded.mobileprovision`文件，用之前准备好的可以签名的描述文件`embedded.mobileprovision`替换掉它。
### 重命名BundleId
如果使用的证书不包含当前`.ipa`的BundleId，那么需要将可执行文件的`BundleId`重命名。

在`Frameworks`同级目录，找到`Info.plist`文件，修改Info.plist文件中的BundleId为符合签名证书的BundleId。

注：由于`.app`包内容中的`Info.Plist`是`binary`格式，所以需要使用`Xcode`打开，或者先转换成`xml`格式，修改完成后再转回`binary`。
### 修改其他内容
如果需要，可以修改文件名或版本号，直接修改`Info.plist`中的值即可。
### 重签名app
使用命令，将app重签名（此时需要指定刚才准备好的`entitlements`文件`entitlements_ori.plist`）：
```shell
codesign -fs "Apple Development: *** (*********)" --entitlements entitlements_ori.plist Payload/NeedReCodeSign.app
```
### 压缩并命名`ipa`
将`Payload`文件夹压缩成`zip`，并命名为`.ipa`后缀。
可以手动，或者使用命令：
```shell
zip -ry ReCodeSigned.ipa Payload
```
注：`zip`命令在传入`Payload`参数时，不要携带路径（否则会把路径信息一起压缩），所以需要在`Payload`同级目录执行。

这样，就获得了重新签名的文件`ReCodeSigned.ipa`，而我们的动态库注入工作，也就到此完成。
# 后记
整个学习使用的过程还是有一点坎坷，所以我才想自己写一篇完整的介绍。

同时，我也编写了一份[脚本](https://github.com/AltairEven/InjectAndReCodeSign)，来自动化处理注入过程，需要的同学可以自取。

另外，github上也有一个工具[IPAPatch](https://github.com/Naituw/IPAPatch)，可以用来处理这个工作。
# 参考资料
- [IOS 非越狱代码注入（Framework）](https://www.jianshu.com/p/0163795f61b7)
- [IPAPatch](https://github.com/Naituw/IPAPatch)
- [iOS包重签名技术知识](https://juejin.cn/post/6844904050228461575)
- [iOS逆向(3)-APP重签名](https://juejin.cn/post/6844903790643003405)
- [ipa动态库的剥离](https://www.jianshu.com/p/5f29e4733687)
- [关于xcode：OSX：更改.framework的路径](https://www.codenong.com/1304239/)