# UI Tips
### 不显示statueBar
```swift
override func prefersStatusBarHidden() -> Bool {
    return true
}
```
### 对约束进行操作
```swift
@IBOutlet var imageViewTopConstraint: NSLayoutConstraint!
imageViewLeadingConstraint.constant = 100
view.layoutIfNeeded()
```
## TableView
### tableView头图片下拉放大效果
```swift
var imageView = UIImageView()

imageView.frame = CGRectMake(0, 0, self.view.frame.width, 180)
imageView.image = UIImage(named: "img")
self.view.addSubview(imageView)//UITableViewContronller
tableView.tableHeaderView = UIView(frame: CGRectMake(0, 0, self.view.frame.width, 180))

override func scrollViewDidScroll(scrollView: UIScrollView) {
        let offY = scrollView.contentOffset.y
        print(offY)
        if offY < 0{
            imageView.frame = CGRectMake(offY/2, offY, self.view.frame.width - offY, 180 - offY)
        }
}
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
### 点击cell，以展开cell显示更多内容
```swift
func addNotesViewToCell(cell: ChecklistItemTableViewCell){
    notesView.heightAnchor.constraintEqualToConstant(notesViewHeight).active = true
    notesView.clipsToBounds = true        
    cell.stackView.addArrangedSubview(notesView)
}
func removeNotesView() {
    if let stackView = notesView.superview as? UIStackView {
        stackView.removeArrangedSubview(notesView)
        notesView.removeFromSuperview()
    }
}
{
    override func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {
        guard let cell = tableView.cellForRowAtIndexPath(indexPath) as?
            ChecklistItemTableViewCell else {
                return
        }
        tableView.beginUpdates()
        
        if cell.stackView.arrangedSubviews.contains(notesView) {
            removeNotesView()
        } else {
            addNotesViewToCell(cell)
            notesTextView.text = checklist.items[indexPath.row].notes
        }
       
        tableView.endUpdates()
    }
}
```
# Controllers

## NavigationController
### 跳转到navigation之前的页面（在有navigationController的视图栈中）
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
## Container Controller
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
## 交互
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
# Objective-C Tips
### ⚠️array has mutated while enumerating
```
    NSMutableArray * selectedDiscountIndex = @[@(1),@(3),@(4),@(5)].mutableCopy;
    [selectedDiscountIndex enumerateObjectsUsingBlock:^(NSNumber*  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        if (obj.integerValue > 3) {
            [selectedDiscountIndex removeObjectAtIndex:idx];
            // 这里到@(4)之后直接跳出，不再遍历@(5)
        }
    }];
    // FAILED！
    selectedDiscountIndex = @[@(1),@(3),@(4),@(5)].mutableCopy;
    // CRASH! 这里遍历@(4)之后，到@(5)会crash
    for (NSNumber* index in selectedDiscountIndex) {
        if (index.integerValue > 3) {
            [selectedDiscountIndex removeObject: index];
        }
    }
    // 结论：除非只modify一次并在修改后break，才可以在遍历中修改，否则奔溃或终止便利
    // 所以：应该遍历selectedDiscountIndex.copy
```
