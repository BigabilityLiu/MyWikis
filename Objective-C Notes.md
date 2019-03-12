# Objective-C Notes
教程合集
* [The basics of Objective-C tutorial series by raywenderlich.com](https://www.youtube.com/playlist?list=PL23Revp-82LLqM6azUAr9at03whFNL9Ld)
* [learnxinyminutes.com](https://learnxinyminutes.com/docs/objective-c/)
## [Property Attributes](https://stackoverflow.com/questions/8927727/objective-c-arc-strong-vs-retain-and-weak-vs-assign)
### 原子性
* `atomic(默认)`它保证你一定可以得到一个确定的数据值，而不是一个混乱的内存空间。它保证当对一个属性进行多线程多进程的读和写操作时，也可以返回一个确定
的值，但是并不确定是修改之前或之后的值。但并<b>不要误认为它是线程安全的</b>，你需要自己用其他方式来保证线程安全，atomic只保证你读的时候，总会返回一个值。
* `non-atomic`不使用原子锁，当你读取它的值时，如果它正在被写入修改，那你会得到一个垃圾信息。但是它的速度比atomic更快。当你经常对一个属性进行访问时，使用nonatomic会可以保证你的性能。当你只在一个线程中读取和修改属性时，也采用它，比如在主线程中访问UI属性。

### 修改权限
* `readwrite(默认)`
* `readonly`没有setter的实现。⚠️如果在.m文件的interface中声明一个属性为`readonly`，则在.m中的implement中不可读取，如果在.h文件的`interface`中申明为`readonly`,但是在.m中的interface（“class-continuation分类”）中重写为`readwrite`，则在impement中也可以修改。

### 内存管理
#### 强指引
* `strong(默认)`为该属性设置新值时，设置方法会先保留新值，并释放旧值，然后把新值设置上。只要该属性不是nil，则该Object不能被deallocated and released。当所有的strong指引都变成nil时，该Object将被deallocated.
* `copy` 与`strong`类似，但是设置方法并不保留新值，而是将其copy来，。常用与像NSString这样自身不可变，但含有可变子类的Object上，来保护其封装性。因为传递给设置方法的新值可能是一个NSMutableString类的实例，此时若是不拷贝字符串，那么该属性可能会在不知情情况下执行可变方法导致更改，所以此时就要拷贝一份不可变的字符串，确保该属性不会被无意间变动。
* `retain`==`strong`。`strong`用于最新的ARC模式下，而`retain`是MRC方式下的方法。
#### 弱指引
* `weak`为该属性设置新值时，既不保留新值，也不释放旧值。该属性是不是nil，该Object都可以释放。常用于delegates、 blocks（weakSelf 
should be used instead of self to avoid memory cycles），同`assign`类似。
* `unsafe_unretained`类似`weak`用于属性（Object）类型，区别在于：当目标对象被摧毁后，`weak`属性会指向nil，而`unsafe_unretained`属性值不会被清空，依然指向被摧毁的目标属性，所以说是不安全的（unsafe）。
#### assign
`assign(默认)`效果与弱指引类似，主要用于纯量类型（scalar type）比如int CGFloat NSInteger；

## Blocks
[fuckingblocksyntax.com](http://fuckingblocksyntax.com/)
### 示例
`TableViewCell.h`
```
@property (nonatomic, copy, nullable) void (^myBlock)(void);
```
`TableViewCell.m`
```
self.myBlock();// call this block;
```
`ViewController.m`
 ```
    cell.myBlock = ^{
        NSLog(@"call this block from cell");
    };
 ```

## KVC & KVO
##### [From Objccn.io](https://objccn.io/issue-7-3/)
##### [facebook/KVOController](https://github.com/facebook/KVOController)

## weakSelf & strongSelf
##### [深入研究 Block 用 weakSelf、strongSelf、@weakify、@strongify 解决循环引用](https://halfrost.com/ios_block_retain_circle/)
##### [stackoverflow self & block](https://stackoverflow.com/questions/20030873/always-pass-weak-reference-of-self-into-block-in-arc)
##### [Raywenderlich Instruments-tutorial](https://www.raywenderlich.com/397-instruments-tutorial-with-swift-getting-started)
### 示例
`Model.h`
```
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
```
#import "Model.h"
@implementation Student
@end
@implementation Teacher
@end
```
`ViewController.m`
```
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
👆此处出现了异常。teacher持有student，student持有study block，block持有student，三方相互持有造成异常<br>
修改如下：
```
    __weak __typeof(teacher) weakTeacher = teacher;
    student.study = ^{
        NSLog(@"my name is = %@",weakTeacher.name);
    };
```
但在study block运行时并不能保证每次teacher都不为空，所以需要修改为以下：
```
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
使用#define时如果有人重新定义了常量值，编译器不会发出警告，从而导致应用程序中的常量值不一致。<br>
局部变量,在.m文件中的使用static const来定义，例如：
 ```
 static const NSString* TestString = @"TestString";
 ```
全局变量，需要.h文件中使用extern来声明，并在.m文件中实现，通常其名称需要加以隔离，通常用类名做前缀。例如：
```
// Person.h
extern const NSString* PersonNameChangedNotification;
// Person.m
const NSString* PersonNameChangedNotification = @"PersonNameChangedNotification";
```
## 第12条: 理解消息转发机制
##### Step1: 动态方法解析
+ （BOOL) resolveInstanceMethod:(SEL) selector
##### Step2: 备援接收者
- (id)forwardingTargetForSelector:(SEL)selector
##### Step3: 完整的消息转发
- (void)forwardInvocation:(NSInvocation*)invocation
##### 例子
EOCAutoDictionary.h
```
@interface EOCAutoDictionary : NSObject
@property (nonatomic, strong) NSString *string;
@property (nonatomic, strong) NSDate *date;
@end
```
EOCAutoDictionary.m
```
#import "EOCAutoDictionary.h"
#import <objc/runtime.h>

@interface EOCAutoDictionary ()
@property (nonatomic, strong) NSMutableDictionary* backingStore;

@end

@implementation EOCAutoDictionary

@dynamic string, date;

- (id)init{
    if (self = [super init]) {
        _backingStore = [NSMutableDictionary new];
    }
    return self;
}

+ (BOOL)resolveInstanceMethod:(SEL)sel{
    NSString *selString = NSStringFromSelector(sel);
    if ([selString hasPrefix:@"set"]) {
        class_addMethod(self, sel, (IMP)autoDictionarySetter, "v@:@");
    }else{
        class_addMethod(self, sel, (IMP)autoDictionaryGetter, "@@:");
    }
    return YES;
}
id autoDictionaryGetter(id self, SEL _cmd){
    EOCAutoDictionary *typeSelf = (EOCAutoDictionary*)self;
    NSMutableDictionary *backingStore = typeSelf.backingStore;
    NSString *key = NSStringFromSelector(_cmd);
    return [backingStore objectForKey:key];
}

void autoDictionarySetter(id self, SEL _cmd, id value){
    EOCAutoDictionary *typeSelf = (EOCAutoDictionary*)self;
    NSMutableDictionary *backingStore = typeSelf.backingStore;
    NSMutableString *key = NSStringFromSelector(_cmd).mutableCopy;
    
    [key deleteCharactersInRange:(NSMakeRange(key.length - 1, 1))];
    [key deleteCharactersInRange:NSMakeRange(0, 3)];
    NSString *lowercaseFirstChar = [[key substringToIndex:1] lowercaseString];
    [key replaceCharactersInRange:NSMakeRange(0, 1) withString:lowercaseFirstChar];
    if (value) {
        [backingStore setObject:value forKey:key];
    }else{
        [backingStore removeObjectForKey:key];
    }
}
@end

```
## 第13条: 方法调配技术
一般来说，只在调试程序时候才需要在运行期修改方法实现，不宜滥用，用多了不宜读懂且难以维护
#### Swizzling 方法替换在Objcetive-C和Swift5中的实现
```
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
```
// Objcetive-C
    Method originalTurnOn = class_getInstanceMethod(mustang.class, @selector(turnOn));
    IMP newIMP =  class_getMethodImplementation(mustang.class, @selector(accelerate));
    method_setImplementation(originalTurnOn, newIMP);//method_getImplementation(newTurnOn));
    [mustang turnOn];
```
