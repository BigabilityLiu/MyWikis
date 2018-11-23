# UI Tips
### Container Controller
父类`FirstViewController.m`
```swift
// 声明
SecondViewController *secondVC;
// ...
- (IBAction)func {
        secondVC = [[SecondViewController alloc] init];
        [self addChildViewController:secondVC];
        CGSize size = self.view.bounds.size;
        secondVC.view.frame = CGRectMake(size.width/4, size.height/2, size.width/2, size.height/2);;
        [self.view addSubview:secondVC.view];
        [secondVC didMoveToParentViewController:self];
        // 可以通过KVO 建立数据链接，当然也可以使用单例共享数据或代理等
        [secondVC addObserver:self
           forKeyPath:@"counter"
              options:NSKeyValueObservingOptionNew context:nil];
}
// 观察方法
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if ([object isEqual:secondVC] && [keyPath isEqualToString:@"counter"]) {
        NSLog(@"%s", __PRETTY_FUNCTION__);
        [self setupData];
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}
- (void)dealloc
{
    [secondVC removeObserver:self forKeyPath:@"counter"];
}
        
```
子类`SecondViewController.h`
```swift
    [self willMoveToParentViewController:nil];
    [self.view removeFromSuperview];
    [self removeFromParentViewController];
```
