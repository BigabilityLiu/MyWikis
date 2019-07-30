# iOS部分
## RunLoop

[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/#more-41710)

#### 是干什么的

一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，**让线程能随时处理事件但并不退出**。

#### 与线程之间的联系

- CFRunLoop 是基于 pthread 来管理的
- 线程和 RunLoop 之间是一一对应的

#### Mode

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

**应用场景举例： Timer与ScrollView**主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为”Common”属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入到顶层的 RunLoop 的 “commonModeItems” 中。”commonModeItems” 被 RunLoop 自动更新到所有具有”Common”属性的 Mode 里去。

#### 应用

##### AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，

第一个 Observer 监视的事件是 Entry(**即将进入Loop**)，其回调内会调用 _objc_autoreleasePoolPush() **创建自动释放池**，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(**准备进入休眠**) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() **释放旧的池并创建新池**；Exit(**即将退出Loop**) 时调用 _objc_autoreleasePoolPop() 来**释放自动释放池**。这个 Observer 优先级最低，保证其释放池子发生在其他所有回调之后。

##### 事件响应

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，当一个硬件事件(触摸/锁屏/摇晃等)发生后这个 Source1 就会触发回调。

##### 手势识别

##### 界面更新

##### 定时器

NSTimer 其实就是 CFRunLoopTimerRef

##### PerformSelecter

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

##### 关于GCD

RunLoop 底层也会用到 GCD 的东西，同时 GCD 提供的某些接口也用到了 RunLoop， 例如调用 dispatch_async(dispatch_get_main_queue(), block) 。dispatch 到其他线程仍然是由 libDispatch 处理的。

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
我们用串行并行描述队列。这就是在描述，该队列里面的所有任务，相互之间在同一时刻，是怎样的运行关系。是指队列内本身的任务运行顺序。
* serial（串行） 某一时刻，只执行一个任务
* concurrent（并行） 可以同时执行多个任务

侧重描述一个函数的执行完成，对其他任务的影响 (既是否任务在等待某个函数完成，然后才可以运行)：

* synchronous（同步） 任务执行完成后reture，（阻塞）
* asynchronous（异步） 不等待任务执行完成，立即reture，（不阻塞当前）
我们还用同步异步，描述某一个任务。比如说任务A是同步执行的。这就是在说，A任务，会阻塞当前任务，直到A结束。这是指不同任务之间的关系，与队列无关，可以是不同队列，也可以是相同队列。

##### [同步锁:如何实现一个文件（属性）的多线程读，单线程写](https://github.com/BigabilityLiu/MyWikis/blob/master/iOS/Objective-C%20Notes.md#%E7%AC%AC41%E6%9D%A1-%E5%A4%9A%E7%94%A8%E6%B4%BE%E5%8F%91%E9%98%9F%E5%88%97%E5%B0%91%E7%94%A8%E5%90%8C%E6%AD%A5%E9%94%81)

## 实践
#### 开发经历中，遇到了什么最难解决的问题，然后怎么克服了

#### Alamofire源码解读（HTTP/RunLoop）

#### tableViewCell里有大图片，怎样处理将网络请求降到最低，效率提升到最高

# [网络](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md)
##### [TCP与UDP区别](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#%E4%BC%A0%E8%BE%93%E5%B1%82)
##### [用户在浏览器输入一个地址后，发生了什么](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#16-%E5%90%84%E7%A7%8D%E5%8D%8F%E8%AE%AE%E4%B8%8E--http-%E5%8D%8F%E8%AE%AE%E7%9A%84%E5%85%B3%E7%B3%BB)
##### [HTTP与HTTPS区别（加签）](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#7-%E7%A1%AE%E4%BF%9D-web-%E5%AE%89%E5%85%A8%E7%9A%84https)


# 算法篇
##### [十种排序算法](http://www.codeceo.com/article/10-sort-algorithm-interview.html#0-tsina-1-10490-397232819ff9a47a7b7e80a40613cfe1)



