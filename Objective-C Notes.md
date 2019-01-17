# Objective-C Notes
教程合集
* [The basics of Objective-C tutorial series by raywenderlich.com](https://www.youtube.com/playlist?list=PL23Revp-82LLqM6azUAr9at03whFNL9Ld)
* [learnxinyminutes.com](https://learnxinyminutes.com/docs/objective-c/)
## [Property Attributes](https://stackoverflow.com/questions/8927727/objective-c-arc-strong-vs-retain-and-weak-vs-assign)
### 原子性
* `atomic(默认)`它保证你一定可以得到一个确定的数据值，而不是一个混乱的内存空间。它保证当对一个变量进行多线程多进程的读和写操作时，也可以返回一个确定
的值，但是并不确定是修改之前或之后的值。但并<b>不要误认为它是线程安全的</b>，你需要自己用其他方式来保证线程安全，atomic只保证你读的时候，总会返回一个值。
* `non-atomic`当你读取它的值时，如果它正在被写入修改，那你会得到一个垃圾信息。但是它的速度比atomic更快。当你经常对一个变量进行访问时，使用nonatomic会
可以保证你的性能。当你只在一个线程中读取和修改变量时，也采用它，比如在主线程中访问UI变量。

### 修改权限
* `readwrite(默认)`
* `readonly`没有setter的实现。⚠️如果在.m文件的interface中声明一个变量为`readonly`，则在.m中的implement中不可读取，如果在.h文件的`interface`中申明为
`readonly`,但是在.m中的interface中重写为`readwrite`，则在impement中也可以修改。

### 内存管理
* `strong(默认)`表示该变量强指引到Object上，只要该变量不是nil，则该Object不能被deallocated and released。当所有的strong指引都变成nil时，
该Object将被deallocated.
* `weak`表示该变量弱指引到Object上，该变量是不是nil，该Object都可以释放。常用于1. UI objects, 2. delegates , 3. blocks（weakSelf 
should be used instead of self to avoid memory cycles）
### 
* `assign(默认)`主要用于C语言简单类型比如int float bool；
* `copy`常用与像string和所有可变的Object上and so on...只有实现了NSCopying协议，并且实现了其中的copyWithZone:方法的对象才能被拷贝。
⚠️`retain(默认)`==`strong`。`weak`约等于`assign`，`assign`用于C语言基本类型，`weak`用于references to Objective-C objects.
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
