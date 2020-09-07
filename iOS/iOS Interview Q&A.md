# iOS部分
## RunLoop

[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/#more-41710)

#### 是干什么的

一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，**让线程能随时处理事件但并不退出**。所以，RunLoop 实际上就是一个对象，**这个对象管理了其需要处理的事件和消息**，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

#### 与线程之间的联系

- CFRunLoop 是基于 pthread 来管理的
- 线程和 RunLoop 之间是一一对应的。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

#### NSRunLoopMode

1. [`NSRunLoopCommonModes`](https://developer.apple.com/documentation/foundation/nsrunloopcommonmodes?language=objc)all run loop modes that have been declared as a member of the set of “common" modes
2. [`NSDefaultRunLoopMode`](https://developer.apple.com/documentation/foundation/nsdefaultrunloopmode?language=objc)The mode to deal with input sources other than(除...之外) [`NSConnection`](https://developer.apple.com/documentation/foundation/nsconnection?language=objc) objects.
3. [`NSEventTrackingRunLoopMode`](https://developer.apple.com/documentation/appkit/nseventtrackingrunloopmode?language=objc)A run loop should be set to this mode when tracking events modally, such as a mouse-dragging loop.
4. [`NSModalPanelRunLoopMode`](https://developer.apple.com/documentation/appkit/nsmodalpanelrunloopmode?language=objc)A run loop should be set to this mode when waiting for input from a modal panel, such as `NSSavePanel` or `NSOpenPanel`.
5. [`UITrackingRunLoopMode`](https://developer.apple.com/documentation/uikit/uitrackingrunloopmode?language=objc)The mode set while tracking in controls takes place.

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

##### Mode item

1. **CFRunLoopSourceRef** 是事件产生的地方。
2. **CFRunLoopTimerRef** 是基于时间的触发器。
3. **CFRunLoopObserverRef** 是观察者，当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。

一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。

**应用场景举例： Timer与ScrollView**

主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为”Common”属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入到顶层的 RunLoop 的 “commonModeItems” 中。”commonModeItems” 被 RunLoop 自动更新到所有具有”Common”属性的 Mode 里去。

#### 应用

##### 事件响应

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

##### 手势识别

##### 界面更新

当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。苹果注册的监听 BeforeWaiting(即将进入休眠)和Exit (即将退出Loop) 事件的Observer，会回调去执行一个很长的函数，这个函数里会遍历所有待处理的 UIView/CAlayer以执行实际的绘制和调整，并更新UI界面。

##### 定时器

NSTimer 其实就是 CFRunLoopTimerRef

##### PerformSelecter

当调用NSObject的performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop中。所以如果当前线程没有RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

##### 关于GCD

RunLoop 底层也会用到 GCD 的东西，同时 GCD 提供的某些接口也用到了RunLoop， 例如调用dispatch_async

当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__() 里执行这个 block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

##### AutoreleasePool

### AutoreleasePool 

* 自动释放池是由 AutoreleasePoolPage 以双向链表的方式实现的
* 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
* 调用 AutoreleasePoolPage::pop 方法会向栈中的对象发送 release 消息

####  Autorelease Pool 进行 Drain 的时机

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，

第一个 Observer 监视的事件是 Entry(**即将进入Loop**)，其回调内会调用 _objc_autoreleasePoolPush() **创建自动释放池**，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(**准备进入休眠**) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() **释放旧的池并创建新池**；Exit(**即将退出Loop**) 时调用 _objc_autoreleasePoolPop() 来**释放自动释放池**。这个 Observer 优先级最低，保证其释放池子发生在其他所有回调之后。

系统在 runloop 中创建的 autoreleaspool 会在**runloop一个event** 结束时进行释放操作。

我们手动创建的 autoreleasepool 会在 **block 执行完成**之后进行 drain 操作。需要注意的是：

* 当 block 以异常（exception）结束时，pool 不会被 drain
* Pool 的 drain 操作会把所有标记为 autorelease 的对象的引用计数减一，但是并不意味着这个对象一定会被释放掉，我们可以在 autorelease pool 中手动 retain 对象，以延长它的生命周期（在 MRC 中）。

#### 手动操作

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

如果不使用 autoreleasepool ，需要在循环结束之后释放 100000000 个字符串，如果使用的话，则会在每次循环结束的时候都进行 release 操作。

#### main.m 中 Autorelease Pool

UIApplicationMain 函数是整个 app 的入口,它自己会创建一个 main run loop，我们大致可以得到下面的结论：

1. main.m 中的 UIApplicationMain 永远不会返回，只有在系统 kill 掉整个 app 时，系统会把应用占用的内存全部释放出来。
2. 因为(1)， UIApplicationMain 永远不会返回，这里的 autorelease pool 也就永远不会进入到释放那个阶段
3. 在 (2) 的基础上，假设有些变量真的进入了 main.m 里面这个 pool（没有被更内层的 pool 捕获），那么这些变量实际上就是被泄露的。这个 autorelease pool 等于是把这种泄露情况给隐藏起来了。
4. UIApplication 自己会创建 main runloop，在 Cocoa 的 runloop 中实际上也是自动包含 autorelease pool 的，**因此 main.m 当中的 pool 可以认为是没有必要的。**

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

**SEL：**类成员方法的指针，方法编号

**IMP：**函数指针，保存了应该执行方法的地址

## 内存管理

#### ARC是什么

在对象的生命周期中自动添加retain、release、autorelease等操作。

#### `property`的作用是什么

实际上@property = 实例变量 + get方法 + set方法

1. 自动合成getter和setter方法
2. 关键词能够传递出相关行为的额外信息

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

## [事件的传递和响应机制](https://www.jianshu.com/p/2e074db792ba)

## 实践
#### TableViewCell里有大图片，怎样处理将网络请求降到最低，效率提升到最高

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

#### 通过子类修改父类私有属性

1. 通过`setValue:(id)value forKey`修改

2. 通过运行时`performSelector:withObject:`方式修改

3. 在子类中创建父类的class extension

4. 在子类在内部声明一个跟父类内部同名的属性

#### NSObject的内存大小

NSObject编译成C++代码后，如下:

```c++
struct NSObject_IMPL {
    Class isa;
};
typedef struct objc_class *Class;
```

其实就是一个指向 `struct objc_class` 结构体类型的指针. 那么也就是说目前我们只发现 `NSObject` 对象对应的结构体只包含一个 `isa` 指针变量 , 一个指针变量在 64 位的机器上大小是 8 个字节。但**所有的OC对象至少为16字节**.

```objective-c
NSObject *lbobjc = [[NSObject alloc] init];
        
NSLog(@"实际需要的内存大小: %zd",class_getInstanceSize([lbobjc class]));
NSLog(@"实际分配的内存大小: %zd",malloc_size((__bridge const void *)(lbobjc)));
// 实际需要的内存大小: 8
// 实际分配的内存大小: 16
```

（在64位机器上）结论如下：

1. OC对象最少占用 `16` 个字节内存，满足 `16` 字节对齐标准 。
2. 属性最终满足 `8` 字节对齐标准 .
3. 当对象中包含属性, 会按属性占用内存开辟空间. 在结构体内存分配原则下自动偏移和补齐 .
4. 可以通过 #pragma pack() 自定义对齐方式 .

#### OC的反射机制

```objective-c
// SEL和字符串转换
NSString *NSStringFromSelector(SEL aSelector);
SEL NSSelectorFromString(NSString *aSelectorName);
// Class和字符串转换
NSString *NSStringFromClass(Class aClass);
Class __nullable NSClassFromString(NSString *aClassName);
// Protocol和字符串转换
NSString *NSStringFromProtocol(Protocol *proto);
Protocol * __nullable NSProtocolFromString(NSString *namestr);
```

```objective-c
// 当前对象是否这个类或其子类的实例
- (BOOL)isKindOfClass:(Class)aClass;
// 当前对象是否是这个类的实例
- (BOOL)isMemberOfClass:(Class)aClass;
// 当前对象是否遵守这个协议
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;
// 当前对象是否实现这个方法
- (BOOL)respondsToSelector:(SEL)aSelector;
```

#### iOS的timer和CADisplayLink的区别

`CADisplayLink`是一个能让我们以和屏幕刷新率相同的频率将内容画到屏幕上的定时器，使用场合相对专一，适合做UI的不停重绘，比如自定义动画引擎或者视频播放的渲染。

`NSTimer`的使用范围要广泛的多，各种需要单次或者循环定时处理的任务都可以使用。

#### CADisplayLink

优势：依托于设备屏幕刷新频率触发事件，所以其触发时间上是最准确的。也是`最适合做UI不断刷新的事件`，过渡相对流畅，无卡顿感。

缺点：

- 由于依托于屏幕刷新频率，如果CPU不堪重负而影响了屏幕刷新，那么我们的触发事件也会受到相应影响。
- selector触发的时间间隔只能是duration的整倍数。
- selector事件执行时间如果大于其触发间隔就会造成掉帧现象。

#### weak是如何实现的，指针自动置为nil的底层实现

[解读objc源码：weak的实现原理](https://www.jianshu.com/p/bcd8239fc1a8)

```objective-c
{
	id obj = NSObject.new;
	id __weak obj_weak = obj;
  // objc_initWeak(&obj_weak,obj);
  // objc_storeWeak(&obj_weak,obj);
  obj = nil;
	// objc_storeWeak(&obj_weak,0);
}
```

Runtime维护了一个Weak表，用于存储指向某个对象的所有Weak指针。Weak表其实是一个哈希表，Key是所指对象（obj）的地址，Value是包含Weak指针（obj_weak）的地址数组；

weak 的实现原理可以概括一下三步：

1、初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。

2、添加引用时：objc_initWeak函数会调用 objc_storeWeak() 函数， objc_storeWeak() 的作用是更新指针指向，创建对应的弱引用表。

3、释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。

#### 为什么会卡顿，如何监控，手动实现监控FPS？如何监控线上的卡顿？

[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

[Tencent/matrix](https://github.com/Tencent/matrix) [微信终端开发团队的专栏](https://cloud.tencent.com/developer/column/1362)

[iOS卡顿监测方案总结](https://juejin.im/post/6844903944867545096#heading-1)

[RunLoop实战：实时卡顿监控](https://juejin.im/post/6844903815682981895#heading-0)

原因：

卡顿就是在应用使用过程中出现界面不响应或者界面渲染粘滞的情况。而应用界面的渲染以及事件响应是在主线程完成的，出现卡顿的原因可以归结为**主线程阻塞**。

在开发过程中，遇到的造成主线程阻塞的原因可能是：

- 主线程在进行大量I/O操作：为了方便代码编写，直接在主线程去写入大量数据；
- 主线程在进行大量计算：代码编写不合理，主线程进行复杂计算；
- 大量UI绘制：界面过于复杂，UI绘制需要大量时间；
- 主线程在等锁：主线程需要获得锁A，但是当前某个子线程持有这个锁A，导致主线程不得不等待子线程完成任务。

使用CADisplayLink实现监控

```objc
@interface ViewController ()
@property (weak, nonatomic) IBOutlet UILabel *fpsLabel;
@property (strong, nonatomic) CADisplayLink* link;
@property (assign, nonatomic) CFTimeInterval lastTimestamp;

@end

@implementation ViewController
  // 每秒钟监控的次数。例如 1秒检测6次
static int monitorTimesPerSecond = 6;
- (void)viewDidLoad {
    [super viewDidLoad];
    self.link = [CADisplayLink displayLinkWithTarget:self selector:@selector(frame)];
    self.link.preferredFramesPerSecond = monitorTimesPerSecond;
    [self.link addToRunLoop:NSRunLoop.mainRunLoop forMode:NSRunLoopCommonModes];
}
- (void)frame {
    if (self.lastTimestamp == 0) {
        self.lastTimestamp = self.link.timestamp;
    } else {
        double timeInterval = self.link.timestamp - self.lastTimestamp;
        self.fpsLabel.text = [NSString stringWithFormat:@"%.0f", 60 / ((double)monitorTimesPerSecond * timeInterval)];
        self.lastTimestamp = self.link.timestamp;
    }
}
```

#### 如何手动监控所有的方法的运行时间？

Hook msg_send

#### NSArray 的底层数据结构

[CFArray](https://developer.apple.com/documentation/corefoundation/cfarray-s28)

[CFMutableArray](https://developer.apple.com/documentation/corefoundation/cfmutablearray-rrk)

#### iOS中的锁

锁用于解决线程争夺资源的问题，一般分为两种，自旋锁(spin)和互斥锁(mutex)。

**互斥锁**可以解释为线程获取锁，发现锁被占用，就向系统申请锁空闲时唤醒他并立刻休眠。os_unfair_lock、@ synchronized、NSLock、NSConditionLock 、NSCondition、NSRecursiveLock

**自旋锁**比较简单，当线程发现锁被占用时，会不断循环判断锁的状态，直到获取。自旋锁的优点在于，因为自旋锁不会引起调用者睡眠，所以不会进行线程调度，CPU时间片轮转等耗时操作。所有如果能在**很短的时间内获得锁，自旋锁的效率远高于互斥锁**。缺点在于，自旋锁一直占用CPU，他在未获得锁的情况下，一直运行－－自旋，所以占用着CPU，如果不能在很短的时 间内获得锁，这无疑会使CPU效率降低。自旋锁不能实现递归调用。OSSpinLock、dispatch_semaphore_t

atomic以前是用的OSSpinLock实现，现在是os_unfair_lock

#### 一个类，如何不让别人访问某些方法

NS_UNAVAILABLE

#### 讲解熟悉的开源库AFNetworking, MJExtension, SDWebImage
#### Runloop，Runtime源码阅读

# [网络](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md)
#### [TCP与UDP区别](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#%E4%BC%A0%E8%BE%93%E5%B1%82)
#### [用户在浏览器输入一个地址后，发生了什么](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#16-%E5%90%84%E7%A7%8D%E5%8D%8F%E8%AE%AE%E4%B8%8E--http-%E5%8D%8F%E8%AE%AE%E7%9A%84%E5%85%B3%E7%B3%BB)
#### [HTTP与HTTPS区别（加签）](https://github.com/BigabilityLiu/MyWikis/blob/master/%E7%AE%97%E6%B3%95:%E7%BD%91%E7%BB%9C/%E5%9B%BE%E8%A7%A3HTTP.md#7-%E7%A1%AE%E4%BF%9D-web-%E5%AE%89%E5%85%A8%E7%9A%84https)


# 算法篇
#### [十种排序算法，重点（归并，堆，快速，插入）](http://www.codeceo.com/article/10-sort-algorithm-interview.html#0-tsina-1-10490-397232819ff9a47a7b7e80a40613cfe1)

#### 递归的缺点
1. 效率低：递归由于是函数调用自身，而函数调用是有时间和空间的消耗的：每一次函数调用，都需要在内存栈中分配空间以保存参数、返回地址以及临时变量，而往栈中压入数据和弹出数据都需要时间。
2. 效率低：递归中很多计算都是重复的。
3. 性能差：调用栈可能会溢出，其实每一次函数调用会在内存栈中分配空间，而每个进程的栈的容量是有限的，当调用的层次太多时，就会超出栈的容量，从而导致栈溢出。

# [git-flow](https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/git-flow)

# 项目

#### 开发经历中，遇到了什么最难解决的问题，然后怎么克服了

#### 项目中的亮点