# Blocks

## Blocks概要

### 什么是Blocks

Blocks是带有自动变量（局部变量）的匿名函数。

## Blocks模式

### 截获自动变量值

```objc
int a= 1;
NSString* fmt = @"original a = %d";
void (^printA)(void) = ^{
    NSLog(fmt, a);
};
printA();
fmt = @"newValue a = %d";
a = 2;
printA();
// 结果：
// original a = 1
// original a = 1
```

Block 语法的表达式使用的是它之前声明的自动变量fmt 和 a。Blocks中，Block 表达式截获所使用的自动变量的值，即保存该自动变量的瞬间值。因为Block 表达式保存了自动变量的值，所以在执行 Block 语法后，即使改写Block中使用的自动变量的值也不会影响Block 执行时自动变量的值。执行结果并不是改写后的值， 而是执行Block 语法时的自动变量的瞬间值。这就是自动变量值的截获。

### _ _block 说明符

自动变量截获只能保存执行Block语法瞬间的值。保存后就不能改写该值。

使用附有_ _block说明附的自动变量，可在Block中赋值，该变量称为_ _block变量。

```objc
__block int a= 1;
__block NSString* fmt = @"original a = %d";
void (^printA)(void) = ^{
   	// 可以赋值修改 a = 2;
    NSLog(fmt, a);
};
printA();
fmt = @"newValue a = %d";
a = 2;
printA();
// 结果：
// original a = 1
// newValue a = 2
```

## Blocks的实现

### Block的实质

block其实就是Objective-C对象

### 截获自动变量值

# Grand Central Dispatch

## GCD概要

### 多线程编程

OS X和iOS的核心XNU内核在发生操作系统啦件时（如毎隔一定时间，唤起系统调用等 情况）会切换执行路径。执行中路径的状态，例如CPU的寄存器等信息保存到各自路径专用的 内存块中，从切换目标路径专用的内存块中，复原CPU寄存器等信息，继续执行切换路径的 CPU命令列。这被称为**“上下文切换”**。

由于使用多线程的程序可以在某个线程和其他线程之间反复多次进行上下文切换，因此看上去就好像1个CPU核能够并列地执行多个线程一样。而且在具有多个CPU核的情况下，就不止“看上去像”了。而是真的提供了多个CPU核并行执行多个线程的技术。
这种利用多线程编程的技术就被称为**“多线程编程”**。

但是，多线程编程实际上是一种易发生各种问题的编程技术。比如多个线程更新相同的资源会导致数据的不一致（数据竞争）、停止等待事件的线程会导致多个线程相互持续等待（死锁）、使用太多线程会消耗大最内存等。

## GCD 的API

### Dispatch Queue

- Serial Dispatch Queue：等待现在执行中的处理结束
- Concurrent Dispatch Queue：不等待现在执行中的处理结束

### dispatch_queue_create

```objc
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);
```

一旦生成Serial Dispatch Queue并追加处理，系统对于一个Serial就会生成并使用一个线程。

生成过多线程，会消耗大量内存，引起大量上下文切换，大幅度降低系统响应性能。

当多个线程更新相同资源会导致数据竞争，所以只在一个线程中更新数据，使用Serial Dispatch Queue，

当想并行执行 **不会发生数据竞争等问题** 的处理时，使用Concurrent Dispatch Queue。因为，不管生成多少，XNU内核只使用有效管理的线程，不会出现Serial那些问题。

```objc
dispatch_queue_t myConcurrentDispatchQueue = dispatch_queue_create("com.example.gcd.MyConcurrentDispatchQueue", DISPATCH_QUEUE_CONCURRENT);
```

生成的Dispatch Queue必须由程序员负责释放。

```objc
dispatch_release(mySerialDispatchQueue);
```

当在dispatch_async中追加Block到Dispatch Queue中时，该Block通过dispatch_retain函数持有Dispatch Queue。一旦Block执行结束后，就通过dispatch_release函数释放该Block持有的Dispatch Queue。

另外在GCD的其他API中，如果名称中有“create”生成的对象，在不使用时，有必要通过dispatch_release函数进行释放。

### Main Dispatch Queue/Global Dispatch Queue

Main Dispatch Queue是在主线程队列，是Serial DispatchQueue。处理的block在主线程的RunLoop中执行。

Global Dispatch Queue是Concurrent DispatchQueue。

### Dispatch Group

#### dispatch_group_notify

```objc
dispatch_queue_t queue =
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); 

dispatch_queue_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0");});
dispatch_group_async(group, queue, A{NSLog(@"blk1");}); 
dispatch_group_async(group, queue, A(NSLog(@"blk2");});
                     
dispatch_group_notify(group,
dispatch_get_main_queue(), ^{NSLog(@"done");});
                     
dispatch_release(group); // 使用结束记得release
```

#### dispatch_group_wait

```objc
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3ull * NSEC_PER_SEC);
long result = dispatch_group_wait(group, time);
if (result == 0) {
	// group全部执行结束
} else {
	// group中某一个还在执行中
}
```

如果指定时间为`DISPATCH_TIME_FOREVER`,意味着永久等待

如果指定时间为`DISPATCH_TIME_NOW`,则不用任何等待即可判断group的处理是否结束。

### dispatch_barrier_async

在并行对列中，将block通过barrier添加到队列中时，barrier会等待队列中所有的block全部执行完毕后，再运行block，此barrier block运行完后，会继续运行之后的所有block

### dispatch_sync

dispatch_sync是将block“同步”追加到队列中，在追加队列结束之前，dispatch_sync函数会一直等待，其实就是简化版的dispatch_group_wait函数。“等待”意味着当前线程停止。

```objc
// 一下两个block都会卡死主线程
dispatch_queue_t mainQueue = dispatch_get_main_queue();
dispatch_sync(mainQueue, ^{
    NSLog(@"hello");
});
dispatch_async(mainQueue, ^{
  dispatch_sync(mainQueue, ^{
      NSLog(@"hello");
  });
});
```

该源代码在主线程中执行block，并等待执行结束，而其实在主线程中正在执行这些源代码，所以无法执行追加到主线程的Block。

在Serial Dispatch Queue中使用dispatch_sync也会有相同问题请注意。

### dispatch_apply

dispatch_apply([想要执行的次数], queue, ^(size_t 当前执行的index) {});

### dispatch_suspend / dispatch_resume

Dispatch Semaphore

```objc
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_semaphore_t semaphore = dispatch_semaphore_create(2);
  dispatch_async(globalQueue, ^{
      // wait 进行减一操作
      long result = dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
      NSLog(@"result = %ld", result);
      sleep(2);
      // signal 进行加一操作
      long result2 = dispatch_semaphore_signal(semaphore);
      NSLog(@"result2 = %ld", result2);

  });
```

### dispatch_once

```objc
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    NSLog(@"Only one");
});
```

### Dispatch IO

进行文件大量读取时，使用dispatch i/o