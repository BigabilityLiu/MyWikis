# iOS部分
## RunLoop

[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/#more-41710)

#### 是干什么的

一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，**让线程能随时处理事件但并不退出**。所以，RunLoop 实际上就是一个对象，**这个对象管理了其需要处理的事件和消息**，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

#### 与线程之间的联系

- CFRunLoop 是基于 pthread 来管理的
- 线程和 RunLoop 之间是一一对应的。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

#### Mode

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

1. **CFRunLoopSourceRef** 是事件产生的地方。
2. **CFRunLoopTimerRef** 是基于时间的触发器。
3. **CFRunLoopObserverRef** 是观察者，当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。

上面的 Source/Timer/Observer 被统称为 **mode item**，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。

**应用场景举例： Timer与ScrollView**

主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为”Common”属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入到顶层的 RunLoop 的 “commonModeItems” 中。”commonModeItems” 被 RunLoop 自动更新到所有具有”Common”属性的 Mode 里去。

#### 应用

##### AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，

第一个 Observer 监视的事件是 Entry(**即将进入Loop**)，其回调内会调用 _objc_autoreleasePoolPush() **创建自动释放池**，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(**准备进入休眠**) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() **释放旧的池并创建新池**；Exit(**即将退出Loop**) 时调用 _objc_autoreleasePoolPop() 来**释放自动释放池**。这个 Observer 优先级最低，保证其释放池子发生在其他所有回调之后。

##### 事件响应

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，当一个硬件事件(触摸/锁屏/摇晃等)发生后，这个 Source1 就会触发回调。随后用mach port 转发给需要的App进程。然后遵循事件响应机制**UIApplication -> UIWindow -> UIView**找到最适合的view进行动作。

##### 手势识别

##### 界面更新

当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。苹果注册的  监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件的Observer，会回调去执行一个很长的函数，这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

##### 定时器

NSTimer 其实就是 CFRunLoopTimerRef

##### PerformSelecter

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

##### 关于GCD

RunLoop 底层也会用到 GCD 的东西，同时 GCD 提供的某些接口也用到了 RunLoop， 例如调用 dispatch_async(dispatch_get_main_queue(), block) 。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

## Runtime

[iOS Runtime详解](https://juejin.im/post/5ac0a6116fb9a028de44d717)

#### 是干什么的

Objective-C 是一个动态语言，这意味着它不仅需要一个编译器，也需要一个**运行时系统来动态得创建类和对象、进行消息传递和转发**。

#### 应用

- 关联对象(Objective-C Associated Objects)**给分类增加属性**
- 方法魔法(**Method Swizzling**)方法添加和替换和**KVO**实现
- **消息转发**(热更新)解决Bug(下方crash之前阻止预防)
- 实现NSCoding的自动归档和自动解档
- 实现字典和模型的自动转换(MJExtension)

#### [消息传递/转发机制](https://github.com/BigabilityLiu/MyWikis/blob/master/iOS/Objective-C%20Notes.md#%E7%AC%AC11%E6%9D%A1-%E7%90%86%E8%A7%A3objc_msgsend%E7%9A%84%E4%BD%9C%E7%94%A8%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E6%9C%BA%E5%88%B6)

#### [crash之前，有什么办法可以阻止预防](https://github.com/BigabilityLiu/MyWikis/blob/master/iOS/Objective-C%20Notes.md#%E5%9C%A8crash%E4%B9%8B%E5%89%8D%E9%98%BB%E6%AD%A2%E9%A2%84%E9%98%B2)

**SEL：** 类成员方法的指针，方法编号

**IMP：** 函数指针，保存了应该执行方法的地址

## 内存管理

#### ARC是什么

在对象的生命周期中自动添加retain、release、autorelease等操作。

#### `property`的作用是什么

实际上@property = 实例变量 + get方法 + set方法

1. 自动合成getter和setter方法
2. 关键词能够传递出相关行为的额外信息

#### [属性](https://github.com/BigabilityLiu/MyWikis/blob/master/iOS/Objective-C%20Notes.md#property-attributes)

### AutoreleasePool 
* 自动释放池是由 AutoreleasePoolPage 以双向链表的方式实现的
* 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
* 调用 AutoreleasePoolPage::pop 方法会向栈中的对象发送 release 消息

#### 用处
Cocoa Touch 的 Runloop 中，每个 runloop circle 中系统都自动加入了 Autorelease Pool 的创建和释放。
当我们需要创建和销毁大量的对象时，使用手动创建的 autoreleasepool 可以有效的避免内存峰值的出现。因为如果不手动创建的话，外层系统创建的 pool 会在整个 runloop circle 结束之后才进行 drain，手动创建的话，会在 block 结束之后就进行 drain 操作。例子：

```swift
for (int i = 0; i < 100000000; i++)
{
    @autoreleasepool
    {
        NSString* string = @"ab c";
        NSArray* array = [string componentsSeparatedByString:string];
    }
}
```
如果不使用 autoreleasepool ，需要在循环结束之后释放 100000000 个字符串，如果 使用的话，则会在每次循环结束的时候都进行 release 操作。
####  Autorelease Pool 进行 Drain 的时机
系统在 runloop 中创建的 autoreleaspool 会在**runloop一个event** 结束时进行释放操作。我们手动创建的 autoreleasepool 会在 **block 执行完成**之后
进行 drain 操作。需要注意的是：

* 当 block 以异常（exception）结束时，pool 不会被 drain
* Pool 的 drain 操作会把所有标记为 autorelease 的对象的引用计数减一，但是并不意味着这个对象一定会被释放掉，我们可以在 autorelease pool 中手动 retain 对象，以延长它的生命周期（在 MRC 中）。

#### main.m 中 Autorelease Pool
UIApplicationMain 函数是整个 app 的入口,它自己会创建一个 main run loop，我们大致可以得到下面的结论：
1. main.m 中的 UIApplicationMain 永远不会返回，只有在系统 kill 掉整个 app 时，系统会把应用占用的内存全部释放出来。
2. 因为(1)， UIApplicationMain 永远不会返回，这里的 autorelease pool 也就永远不会进入到释放那个阶段
3. 在 (2) 的基础上，假设有些变量真的进入了 main.m 里面这个 pool（没有被更内层的 pool 捕获），那么这些变量实际上就是被泄露的。这个 autorelease pool 等于是把这种泄露情况给隐藏起来了。
4. UIApplication 自己会创建 main runloop，在 Cocoa 的 runloop 中实际上也是自动包含 autorelease pool 的，**因此 main.m 当中的 pool 可以认为是没有必要的。** 

## 多线程
我们用串行并行描述队列。这就是在描述，该队列里面的所有任务，相互之间在同一时刻，是怎样的运行关系。是指队列内本身的任务运行顺序。
* serial（串行） 某一时刻，只执行一个任务
* concurrent（并行） 可以同时执行多个任务

侧重描述一个函数的执行完成，对其他任务的影响 (既是否任务在等待某个函数完成，然后才可以运行)：

* synchronous（同步） 任务执行完成后reture，（阻塞）
* asynchronous（异步） 不等待任务执行完成，立即reture，（不阻塞当前）
我们还用同步异步，描述某一个任务。比如说任务A是同步执行的。这就是在说，A任务，会阻塞当前任务，直到A结束。这是指不同任务之间的关系，与队列无关，可以是不同队列，也可以是相同队列。

##### [同步锁:如何实现一个文件（属性）的多线程读，单线程写](https://github.com/BigabilityLiu/MyWikis/blob/master/iOS/Objective-C%20Notes.md#%E7%AC%AC41%E6%9D%A1-%E5%A4%9A%E7%94%A8%E6%B4%BE%E5%8F%91%E9%98%9F%E5%88%97%E5%B0%91%E7%94%A8%E5%90%8C%E6%AD%A5%E9%94%81)

### [事件的传递和响应机制](https://www.jianshu.com/p/2e074db792ba)

## 实践
#### 开发经历中，遇到了什么最难解决的问题，然后怎么克服了

#### Alamofire源码解读（HTTP/RunLoop）

#### tableViewCell里有大图片，怎样处理将网络请求降到最低，效率提升到最高

#### UIImageView高性能添加圆角
1.最简单的图片圆角设置:
```objective-c
self.imageView.layer.cornerRadius=50;
self.imageView.layer.masksToBounds=YES;
```
2.设置Rasterize光栅化处理
```objective-c
self.imageView.layer.cornerRadius=50;
self.imageView.clipsToBounds=YES;
self.imageView.layer.shouldRasterize = YES;
self.imageView.layer.rasterizationScale=[UIScreen mainScreen].scale;
```
* 启用shouldRasterize可改善离屏渲染（指的是GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。）
* 由于启用shouldRasterize得到的图像会被缓存起来。会将图层Layer不同位图合并成一张位图放在缓存区，不会不断的进行图片渲染。这大大减少了GPU的负担。
* 
如果有很多的子图层或者有复杂的效果应用，开启shouldRasterize就会比重绘所有事务的所有帧划得来得多。
* 但是光栅化原始图像需要时间，而且还会消耗额外的内存。如果layer内容经常发生变化，会导致多次光栅化合成，浪费更多性能。所以需要根据实际情况取舍。

3. Core Graphic 绘制圆角图片（推荐）。
```objective-c
UIGraphicsBeginImageContextWithOptions(self.imageView.bounds.size, NO, [UIScreen mainScreen].scale);
// UIBezierPath贝塞尔曲线绘制裁剪出一个圆形
[[UIBezierPath bezierPathWithRoundedRect:self.imageView.bounds
                            cornerRadius:50] addClip];
//图片在设置的圆形里面进行绘制
[[UIImage imageNamed:@"FlyElephant.jpg"] drawInRect:self.imageView.bounds];
//获取图片
self.imageView.image = UIGraphicsGetImageFromCurrentImageContext();
//结束绘制
UIGraphicsEndImageContext();
```
####  UIView和CALayer
* UIView接受并处理事件

* Layer提供内容的绘制和显示

#### OC如何实现多继承？

A类如何继承B类和C类

1. 消息转发：有三个阶段：动态方法解析、快速消息转发、标准消息转发，在第2或3阶段，把消息转发到B类或C类
2. protocol、delegate：将A类需要继承的方法以及**属性**在ClassB和ClassC中各自声明一份协议，A类遵守这两份协议，同时在C类中实现协议中的方法以及属性
3. Category :在A类新建一个Category，实现B、C类中的方法，或者使用objc_setAssociatedObject动态添加属性

#### autoreleasepool 什么时候释放 在什么场景下使用，iOS原生的使用场景
#### weak指针自动置为nil的底层实现

#### 通过子类修改父类私有属性

1. 通过`setValue:(id)value forKey`修改
2. 通过运行时`performSelector:withObject:`方式修改
3. 在子类中创建父类的class extension
4. 在子类在内部声明一个跟父类内部同名的属性

#### 讲解熟悉的开源库AFNetworking，MJExtension

# [网络](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md)
##### [TCP与UDP区别](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#%E4%BC%A0%E8%BE%93%E5%B1%82)
##### [用户在浏览器输入一个地址后，发生了什么](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#16-%E5%90%84%E7%A7%8D%E5%8D%8F%E8%AE%AE%E4%B8%8E--http-%E5%8D%8F%E8%AE%AE%E7%9A%84%E5%85%B3%E7%B3%BB)
##### [HTTP与HTTPS区别（加签）](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#7-%E7%A1%AE%E4%BF%9D-web-%E5%AE%89%E5%85%A8%E7%9A%84https)


# 算法篇
##### [十种排序算法，重点（快速，插入，归并）](http://www.codeceo.com/article/10-sort-algorithm-interview.html#0-tsina-1-10490-397232819ff9a47a7b7e80a40613cfe1)

# [git-flow](https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/git-flow)

#### 递归的缺点
1. 效率低：递归由于是函数调用自身，而函数调用是有时间和空间的消耗的：每一次函数调用，都需要在内存栈中分配空间以保存参数、返回地址以及临时变量，而往栈中压入数据和弹出数据都需要时间。
2. 效率低：递归中很多计算都是重复的。
3. 性能差：调用栈可能会溢出，其实每一次函数调用会在内存栈中分配空间，而每个进程的栈的容量是有限的，当调用的层次太多时，就会超出栈的容量，从而导致栈溢出。


