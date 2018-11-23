# UI Tips
### Container Controller
父类`FirstViewController.m`
```swift
        SecondViewController *secondVC = [[SecondViewController alloc] init];
        [self addChildViewController:secondVC];
        CGSize size = self.view.bounds.size;
        secondVC.view.frame = CGRectMake(size.width/4, size.height/2, size.width/2, size.height/2);;
        [self.view addSubview:secondVC.view];
        [secondVC didMoveToParentViewController:self];
```
子类`SecondViewController.h`
```swift
    [self willMoveToParentViewController:nil];
    [self.view removeFromSuperview];
    [self removeFromParentViewController];
```
