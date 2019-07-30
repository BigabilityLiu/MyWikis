# Objective-C Notes
教程合集
* [The basics of Objective-C tutorial series by raywenderlich.com](https://www.youtube.com/playlist?list=PL23Revp-82LLqM6azUAr9at03whFNL9Ld)
* [learnxinyminutes.com](https://learnxinyminutes.com/docs/objective-c/)
## [Property Attributes](https://stackoverflow.com/questions/8927727/objective-c-arc-strong-vs-retain-and-weak-vs-assign)
**原子性**

* `atomic(默认)`它保证你一定可以得到一个确定的数据值，而不是一个混乱的内存空间。它保证当对一个属性进行多线程多进程的读和写操作时，也可以返回一个确定
的值，但是并不确定是修改之前或之后的值。但并<b>不要误认为它是线程安全的</b>，你需要自己用其他方式来保证线程安全，atomic只保证你读的时候，总会返回一个值。
* `non-atomic`不使用原子锁，当你读取它的值时，如果它正在被写入修改，那你会得到一个垃圾信息。但是它的速度比atomic更快。当你经常对一个属性进行访问时，使用nonatomic会可以保证你的性能。当你只在一个线程中读取和修改属性时，也采用它，比如在主线程中访问UI属性。

**修改权限**

* `readwrite(默认)`
* `readonly`没有setter的实现。⚠️如果在.m文件的interface中声明一个属性为`readonly`，则在.m中的implement中不可读取，如果在.h文件的`interface`中申明为`readonly`,但是在.m中的interface（“class-continuation分类”）中重写为`readwrite`，则在impement中也可以修改。

**强指引**

* `strong(默认)`为该属性设置新值时，设置方法会先retain保留新值，并release释放旧值，然后把新值设置上。只要该属性不是nil，则该Object不能被deallocated and released。当所有的strong指引都变成nil时，该Object将被deallocated.
* `copy` 与`strong`类似，但是设置方法并不保留新值，而是将其copy来，。常用与像NSString这样自身不可变，但含有可变子类的Object上，来保护其封装性。因为传递给设置方法的新值可能是一个NSMutableString类的实例，此时若是不拷贝字符串，那么该属性可能会在不知情情况下执行可变方法导致更改，所以此时就要拷贝一份不可变的字符串，确保该属性不会被无意间变动。
* `retain`==`strong`。`strong`用于最新的ARC模式下，而`retain`是MRC方式下的方法。

**弱指引**

* `weak`为该属性设置新值时，既不保留新值，也不释放旧值。该属性是不是nil，该Object都可以释放。常用于delegates、 blocks（weakSelf 
should be used instead of self to avoid memory cycles），同`assign`类似。
* `unsafe_unretained`类似`weak`用于属性（Object）类型
* `assign(默认)`效果与弱指引类似，主要用于纯量类型（scalar type）比如int CGFloat NSInteger；
也可以用于Object类型，
**区别在于** 当目标对象被摧毁后，`weak`属性会指向nil，而`unsafe_unretained`和`assign`属性值不会被清空，依然指向被摧毁的目标属性，所以说是不安全的（unsafe）。

## Blocks
使用方法[fuckingblocksyntax.com](http://fuckingblocksyntax.com/)

**示例**

`TableViewCell.h`

```objective-c
@property (nonatomic, copy, nullable) void (^myBlock)(void);
```
`TableViewCell.m`
```swift
self.myBlock();// call this block;
```
`ViewController.m`

 ```swift
    cell.myBlock = ^{
        NSLog(@"call this block from cell");
    };
 ```

## KVC & KVO
[From Objccn.io](https://objccn.io/issue-7-3/)

[facebook/KVOController](https://github.com/facebook/KVOController)

## weakSelf & strongSelf
[深入研究 Block 用 weakSelf、strongSelf、@weakify、@strongify 解决循环引用](https://halfrost.com/ios_block_retain_circle/)

[stackoverflow self & block](https://stackoverflow.com/questions/20030873/always-pass-weak-reference-of-self-into-block-in-arc)

[Raywenderlich Instruments-tutorial](https://www.raywenderlich.com/397-instruments-tutorial-with-swift-getting-started)

示例

`Model.h`

```objective-c
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

typedef void(^Study)(void);
@interface Student : NSObject
@property (nonatomic, copy) NSString* name;
@property (nonatomic, copy) Study study;
@end

@interface Teacher : NSObject
@property (copy , nonatomic) NSString *name;
@property (strong, nonatomic) Student *student;
@end
NS_ASSUME_NONNULL_END
```
`Model.m`
```objective-c
#import "Model.h"
@implementation Student
@end
@implementation Teacher
@end
```
`ViewController.m`
```swift
- (void)viewDidLoad {
    [super viewDidLoad];
    
    Student *student = [[Student alloc]init];
    Teacher *teacher = [[Teacher alloc]init];
    
    teacher.name = @"i'm teacher";
    teacher.student = student;
    
    student.name = @"halfrost";
    student.study = ^{
        NSLog(@"my name is = %@",strongTeacher.name);
    };
    student.study();
}
@end
```
👆此处出现了异常。teacher持有student，student持有study block，block持有student，三方相互持有造成异常

修改如下：

```objective-c
    __weak __typeof(teacher) weakTeacher = teacher;
    student.study = ^{
        NSLog(@"my name is = %@",weakTeacher.name);
    };
```
但在study block运行时并不能保证每次teacher都不为空，所以需要修改为以下：
```objective-c
    __weak __typeof(teacher) weakTeacher = teacher;
    student.study = ^{
        __strong __typeof(weakTeacher) strongTeacher = weakTeacher;
        if (strongTeacher) {
            NSLog(@"my name is = %@",strongTeacher.name);
        }
    };
    teacher = nil;//假设在执行block前出现意外
    student.study();
```
strongSelf的目的是因为一旦进入block执行，不允许self在这个执行过程中释放。block执行完后这个strongSelf 会自动释放，不会存在循环引用问题。但是依然需要判断strongSelf是否为空，因为strongSelf只能保证在函数内即block内不为空，不能保证外部情况。
# Effective Objective-C 读书笔记
## 第2条：在类的头文件中尽量少引入其他头文件
应该在.h文件中尽量使用`@class XXX;`引入类，在.m文件中需要用到时，再使用`#import "XXX".h` 引入头文件,原因：
1. 使用`#import`会引入该类中的所有内容，增加编译时间
2. 两个类头文件中都使用`#import`引入对方类头文件，会导致其中一个类无法编译
## 第4条：多用类型常量，少用#define预处理指令
使用#define时如果有人重新定义了常量值，编译器不会发出警告，从而导致应用程序中的常量值不一致.

局部变量,在.m文件中的使用static const来定义，例如：

 ```objective-c
 static const NSString* TestString = @"TestString";
 ```
全局变量，需要.h文件中使用extern来声明，并在.m文件中实现，通常其名称需要加以隔离，通常用类名做前缀。例如：
```objective-c
// Person.h
extern const NSString* PersonNameChangedNotification;
// Person.m
const NSString* PersonNameChangedNotification = @"PersonNameChangedNotification";
```
## 第11条: 理解objc_msgSend的作用（消息传递机制）
在Objectve-C中，如果向某对象传递消息，那就会使用动态绑定机制来决定需要调用的方法。在底层，所有的方法都是普通的C语言函数，然后对象接收到消息之后，究竟调用哪个方法则完全于运行期决定，甚至可以在运行进行时改变，这些特性是的Objective-C成为一门真正的动态语言。给对象发送信息可以这样写：
```objective-c
id returnValue = [someObject messageName:parameter];
```
本例中，someObject叫做“接收者”，messageName叫做“选择子”。选择子与参数合起来叫“消息”。编译器看到此消息后，将其转换成一条标准的C语言函数调用，该很熟乃是消息传递机制中的核心函数，叫做objc_msgSend，其“原型”如下：
```objective-c
void objc_msgSeng(id self, SEL cmd, ...)
```
第一个参数表示接受者，第二个参数表示选择子，后续参数就是消息中的那些参数，其顺序不变。选择子指的是方法的名字。编译器会把刚才例子中的方法转换为如下的函数:
```objective-c
id returnValue = objc_msgSend(someObject, @selector(messageName:), parameter);
```
**该方法需要在接收者所属的类中搜寻其“方法列表”，如果能找到与其选择子名称相符的方法，就跳至其实现代码，若找不到，就沿着继承体系继续向上查找，等找到合适的方法之后再跳转。如果最终还是找不到相府的方法，那就执行“消息转发”操作。**

## 第12条: 理解消息转发机制
Step1: 动态方法解析（给个机会让类添加这个来实现这个函数）

```objective-c
(BOOL)resolveInstanceMethod:(SEL) selector
```

Step2: 备援接收者（让别的对象去执行这个函数）

```objective-c
(id)forwardingTargetForSelector:(SEL)selector
```

Step3: 完整的消息转发（灵活的将目标函数以其他形式执行）

```objective-c
(void)forwardInvocation:(NSInvocation*)invocation
```

#### 在crash之前，阻止预防

在经过上述的消息传递和转发后，如果都不中，调用doesNotRecognizeSelector抛出异常。

[修改三种方法的实现](https://blog.csdn.net/qian521kun521/article/details/90412263)

我们选择第二个forwardingTargetForSelector来做处理。原因如下：

1. resolveInstanceMethod 需要在类的本身上动态添加它本身不存在的方法，这些方法对于该类本身来说冗余的
2. forwardInvocation可以通过NSInvocation的形式将消息转发给多个对象，但是其开销较大，需要创建新的NSInvocation对象，并且forwardInvocation的函数经常被使用者调用，来做多层消息转发选择机制，不适合多次重写
3. forwardingTargetForSelector可以将消息转发给一个对象，开销较小，并且被重写的概率较低，适合重写

选择了forwardingTargetForSelector之后，可以将NSObject的该方法重写，做以下几步的处理：

1.动态创建一个自定义类

2.动态为自定义类添加对应的Selector，用一个通用的返回0的函数来实现该SEL的IMP

3.将消息直接转发到这个自定义类对象上。

## 第13条: 方法调配技术
一般来说，只在调试程序时候才需要在运行期修改方法实现，不宜滥用，用多了不宜读懂且难以维护

**Swizzling 方法替换在Objcetive-C和Swift5中的实现**

```swift
// Swift5
import UIKit
import Foundation
// 继承自NSObject
class Car: NSObject{
    let name: String
    init(name: String) {
        self.name = name
    }
    // 表明 dynamic
    @objc dynamic func run(){
        print(name + " running")
    }
    @objc func walk(){
        print(name + " walk")
    }
}

let mustang = Car.init(name: "Mustang")
mustang.run()
let m1 = class_getInstanceMethod(Car.self, #selector(mustang.run))
let m2 = class_getInstanceMethod(Car.self, #selector(mustang.walk))
if let m1 = m1, let m2 = m2{
    method_exchangeImplementations(m1, m2)
    mustang.run()
}else{
    print("error")
}

```
```swift
// Objcetive-C
    Method originalTurnOn = class_getInstanceMethod(mustang.class, @selector(turnOn));
    IMP newIMP =  class_getMethodImplementation(mustang.class, @selector(accelerate));
    method_setImplementation(originalTurnOn, newIMP);//method_getImplementation(newTurnOn));
    [mustang turnOn];
```

## 第41条: 多用派发队列，少用同步锁

如果要多个线程执行同一份代码，有时会出问题，通常情况下要用锁来实现某种同步机制。

**方法一：同步块 synchronization block**

```objective-c
- (void)synchronizedMethod{
  @synchronized(self) {
    // Safe code
  }
}
```

它可以保证每个对象实例都不受干扰的运行synchronizedMethod，然而如果滥用@synchronized(self) 则会降低代码效率，因为共用用一个锁的那些同步块，都必须按照顺序执行。若是在self对象上频繁加锁，那么程序可能要等另一段与此无关的代码执行完，才能继续执行当前代码。

属性就是做成**原子的atomic**修饰就可以做到,但是不能说是线程安全的。

方法二： NSLock对象

```objective-c
_lock = [[NSLock alloc] init];
- (void)synchronizedMethod{
	[_lock lock];
	// Safe code
  [_lock unlock];
  }
}
```

**缺陷：**极端情况下会导致死锁，另外效率也不高。

**代替方案：使用GCD “串行同步队列”** serial synchronization queue ，get和set方法都在一个队列中，即可保证数据同步。

```objective-c
_syncQueue = dispatch_queue_create("com.xxx.syncqueue", NULL);
- (NSString*)someString{
  	__block NSString* localSomeString;
  	dispatch_sync(_syncQueue, ^{
				localSomeString = _someString;
  	});
    return localSomeString;
}
- (void)setSomeString:(NSString*)someString{
  	dispatch_sync(_syncQueue, ^{
      	_someString = someString;
    });
}
```

set方法和get方法都安排在序列化的队列中执行，所有针对属性的访问操作就都同步了。

**进一步优化：把set方法同步派发改为异步派发**。set方法并不一定非要是同步的。因为set方法并不用返回值，所以set方法可以改成如下：

```objective-c
- (void)setSomeString:(NSString*)someString{
  	dispatch_async(_syncQueue, ^{
      	_someString = someString;
    });
}
```

弊端：因为执行异步派发时需要拷贝块，如果拷贝块所用的时间超过执行块所用的时间，则会比原来更慢。

**并发队列：** 并行执行多个get方法，而get方法和set方法之间不能并发执行。

```objective-c
_syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
- (NSString*)someString{
  	__block NSString* localSomeString;
  	dispatch_sync(_syncQueue, ^{
				localSomeString = _someString;
  	});
    return localSomeString;
}
- (void)setSomeString:(NSString*)someString{
  	dispatch_sync(_syncQueue, ^{
      	_someString = someString;
    });
}
```

使用栅栏barrier让set方法单独执行

```objective-c
- (void)setSomeString:(NSString*)someString{
  	dispatch_barrier_async(_syncQueue, ^{
      	_someString = someString;
    });
}
```

![](/Users/liuxudong/Documents/GitHub/MyWikis/resource/oc_1.png)
