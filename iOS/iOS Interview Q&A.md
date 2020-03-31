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

#### [属性](https://github.com/BigabilityLiu/MyWikis/blob/master/iOS/Objective-C%20Notes.md#property-attributes)

## 多线程
##### [同步锁:如何实现一个文件（属性）的多线程读，单线程写](https://github.com/BigabilityLiu/MyWikis/blob/master/iOS/Objective-C%20Notes.md#%E7%AC%AC41%E6%9D%A1-%E5%A4%9A%E7%94%A8%E6%B4%BE%E5%8F%91%E9%98%9F%E5%88%97%E5%B0%91%E7%94%A8%E5%90%8C%E6%AD%A5%E9%94%81)

## 实践

#### Alamofire源码解读（HTTP/RunLoop）

#### tableViewCell里有大图片，怎样处理将网络请求降到最低，效率提升到最高


# [网络](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md)
##### [TCP与UDP区别](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#%E4%BC%A0%E8%BE%93%E5%B1%82)
##### [用户在浏览器输入一个地址后，发生了什么](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#16-%E5%90%84%E7%A7%8D%E5%8D%8F%E8%AE%AE%E4%B8%8E--http-%E5%8D%8F%E8%AE%AE%E7%9A%84%E5%85%B3%E7%B3%BB)
##### [HTTP与HTTPS区别（加签）](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#7-%E7%A1%AE%E4%BF%9D-web-%E5%AE%89%E5%85%A8%E7%9A%84https)


# 算法篇
##### [十种排序算法](http://www.codeceo.com/article/10-sort-algorithm-interview.html#0-tsina-1-10490-397232819ff9a47a7b7e80a40613cfe1)



