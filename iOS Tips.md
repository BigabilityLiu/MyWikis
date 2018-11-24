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
### 不显示statueBar
```swift
    override func prefersStatusBarHidden() -> Bool {
        return true
    }
```
### 转场，跳转到之前的页面（在有navigationController的视图栈中）
```swift 
let vc = self.navigationController!.viewControllers[0]
self.navigationController?.popToViewController(vc, animated: true)
//for vc in self.navigationController!.viewControllers{
//      if vc.isKindOfClass(XXXVC){
//              self.navigationController?.popToViewController(vc, animated: true)
//      }
//}
```
### navigationbar常见操作
```swift
self.navigationController?.navigationBar.barTintColor = UIColor.fwNavigationBarColor()
self.navigationController?.navigationBar.tintColor = UIColor.whiteColor()
self.navigationController?.navigationBar.titleTextAttributes = NSDictionary(object:UIColor.whiteColor(),forKey:NSForegroundColorAttributeName) as? [String : AnyObject]
//去除转场后下一级的白雾效果
self.navigationController?.navigationBar.translucent = false
```
### tableView头图片下拉放大效果
```swift
var imageView = UIImageView()
```
```swift
        imageView.frame = CGRectMake(0, 0, self.view.frame.width, 180)
        imageView.image = UIImage(named: "img")
        self.view.addSubview(imageView)//UITableViewContronller
        tableView.tableHeaderView = UIView(frame: CGRectMake(0, 0, self.view.frame.width, 180))
```
```swift
override func scrollViewDidScroll(scrollView: UIScrollView) {
        let offY = scrollView.contentOffset.y
        print(offY)
        if offY < 0{
            imageView.frame = CGRectMake(offY/2, offY, self.view.frame.width - offY, 180 - offY)
        }
    }
```
### 对约束进行操作
```swift
@IBOutlet var imageViewTopConstraint: NSLayoutConstraint!
imageViewLeadingConstraint.constant = 100
view.layoutIfNeeded()
```
### 滑动tableView时隐藏navigationController
```swift
        self.navigationController?.hidesBarsOnSwipe = true
```
当你的tableView需要有排序功能时，拖动tableViewCell移动的Pan手势和滑动隐藏的手势会重合，所以需要做
```swift
        if tableView.isEditing {
            self.navigationController?.barHideOnSwipeGestureRecognizer.cancelsTouchesInView = false
        }else{
            self.navigationController?.barHideOnSwipeGestureRecognizer.cancelsTouchesInView = true
        }
```
### view全部停止编辑
```swift
view.endEditing(true)
```
### 键盘弹出时监控是否挡住文本框并编辑
```
var editingTextField: UITextField!

//注册通知，监控键盘弹出消失事件
        NSNotificationCenter.defaultCenter().addObserver(self, selector: #selector(AddSerChangeMoney.keyboardWillShow(_:)), name: UIKeyboardWillShowNotification, object: nil)
        NSNotificationCenter.defaultCenter().addObserver(self, selector: #selector(AddSerChangeMoney.keyboardWillHide(_:)), name: UIKeyboardWillHideNotification, object: nil)

func textFieldShouldBeginEditing(textField: UITextField) -> Bool {
        editingTextField = textField
        return true
    }

func keyboardWillShow(notification: NSNotification){
        guard editingTextField != nil else { return }
        let info: NSDictionary = notification.userInfo!
        let keyboardSize = info.objectForKey(UIKeyboardFrameEndUserInfoKey)?.CGRectValue().size
        let frame = editingTextField.frame
        let y = frame.origin.y + frame.size.height - (self.view.frame.size.height - keyboardSize!.height)
        let animationDuration: NSTimeInterval = 0.3
        UIView.beginAnimations("ResizeView", context: nil)
        UIView.setAnimationDuration(animationDuration)
        if y > 0 {
            self.view.frame = CGRectMake(0, -y, self.view.frame.size.width, self.view.frame.size.height)
        }
        UIView.commitAnimations()
    }
    func keyboardWillHide(notification: NSNotification){
        let animationDuration: NSTimeInterval = 0.3
        UIView.beginAnimations("ResizeView", context: nil)
        UIView.setAnimationDuration(animationDuration)
        self.view.frame = CGRectMake(0, 0, self.view.frame.size.width, self.view.frame.size.height)
        UIView.commitAnimations()
    }
```
