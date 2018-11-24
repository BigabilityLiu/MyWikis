### 转场动画
```swift
let ca = CATransition()
ca.type = "cube"//"push"...
ca.subtype = kCATransitionFromLeft//"FromRight"...
ca.duration = 0.5
imageView.layer.addAnimation(ca, forKey: nil)
```
### 普通动画
```swift
heading.center.x  -= view.bounds.width

UIView.animate(withDuration: 0.5) {
    self.heading.center.x += self.view.bounds.width
}
```
### 手动创造一个类立方体动画
```swift
enum AnimationDirection: Int {
    case positive = 1
    case negative = -1
}

func cubeTransition(label: UILabel , text: String, direction: AnimationDirection){
        let auxLabel = UILabel(frame: label.frame)
        auxLabel.text = text
        auxLabel.font = label.font
        auxLabel.textAlignment = label.textAlignment
        auxLabel.textColor = label.textColor
        auxLabel.backgroundColor = label.backgroundColor
        
        let auxLabelOffset = CGFloat(direction.rawValue) * label.frame.height/2
        
        auxLabel.transform = CGAffineTransform(scaleX: 1, y: 0.05).concatenating(CGAffineTransform(translationX: 0, y: auxLabelOffset))
        
        label.superview?.addSubview(auxLabel)
        
        UIView.animate(withDuration: 0.5, delay: 0, options: .curveEaseOut, animations: { 
            auxLabel.transform = .identity
            label.transform = CGAffineTransform(scaleX: 1, y: 0.05).concatenating(CGAffineTransform(translationX: 0, y: -auxLabelOffset))
        }) { (_) in
            label.text = auxLabel.text
            label.transform = .identity
            
            auxLabel.removeFromSuperview()
        }
    }

```
### 帧动画
```swift
func planeDepart(){
        let originalCenter = planeImage.center
        //withDuration 总共时长3s,delay 延迟0s发生
        UIView.animateKeyframes(withDuration: 3, delay: 0, options: [], animations: {
            //动画从0s开始，持续时间3s*0.25
            UIView.addKeyframe(withRelativeStartTime: 0, relativeDuration: 0.25, animations: {
                self.planeImage.center.x += 80
                self.planeImage.center.y -= 10
            })
            //动画从0.1 * 3s开始，持续时间3s*0.4
            UIView.addKeyframe(withRelativeStartTime: 0.1, relativeDuration: 0.4, animations: { 
                self.planeImage.transform = CGAffineTransform(rotationAngle: -.pi/8)
            })
            //动画从0.25 * 3s开始，持续时间3s*0.25
            UIView.addKeyframe(withRelativeStartTime: 0.25, relativeDuration: 0.25) {
                self.planeImage.center.x += 100.0
                self.planeImage.center.y -= 50.0
                self.planeImage.alpha = 0.0
            }
            UIView.addKeyframe(withRelativeStartTime: 0.51, relativeDuration: 0.01, animations: { 
                self.planeImage.transform = .identity
                self.planeImage.center = CGPoint(x: 0, y: originalCenter.y)
            })
            UIView.addKeyframe(withRelativeStartTime: 0.55, relativeDuration: 0.45, animations:{ self.planeImage.alpha = 1
                self.planeImage.center = originalCenter
            })
        }) { (_) in
        }
    }
```
### 约束动画
```swift 
@IBOutlet weak var menuHeightConstraint: NSLayoutConstraint!
@IBAction func actionToggleMenu(_ sender: AnyObject) {
        isMenuOpen = !isMenuOpen
        //view的约束是在superView中
        titleLabel.superview?.constraints.forEach { constraint in
            if constraint.firstItem === titleLabel && constraint.firstAttribute == .centerX {
                constraint.constant = isMenuOpen ? -100.0 : 0.0
                return
            }
            if constraint.identifier == "TitleCenterY" {
                constraint.isActive = false
                //创建约束
                let newConstraint = NSLayoutConstraint(
                    item: titleLabel,
                    attribute: .centerY,
                    relatedBy: .equal,
                    toItem: titleLabel.superview!,
                    attribute: .centerY,
                    multiplier: isMenuOpen ? 0.67 : 1.0,
                    constant: 5.0)
                newConstraint.identifier = "TitleCenterY"
                newConstraint.isActive = true
                
                return
            }
        }
        
        menuHeightConstraint.constant = isMenuOpen ? 200.0 : 60.0
        titleLabel.text = isMenuOpen ? "Select Item" : "Packing List"
        
        UIView.animate(withDuration: 1.0, delay: 0.0, usingSpringWithDamping: 0.4,
                       initialSpringVelocity: 10.0, options: .curveEaseIn,
                       animations: {
                        let angle: CGFloat = self.isMenuOpen ? .pi / 4 : 0.0
                        self.buttonMenu.transform = CGAffineTransform(rotationAngle: angle)
                        //设置好约束后使生效都是使用一下方法
                        self.view.layoutIfNeeded()
        },
                       completion: nil
        )
        
        
        if isMenuOpen {
            slider = HorizontalItemList(inView: view)
            //在原view中设置一个方法，在此文件中定义方法的内容
            slider.didSelectItem = {index in
                print("add \(index)")
                self.items.append(index)
                self.tableView.reloadData()
                self.actionToggleMenu(self)
            }
            self.titleLabel.superview?.addSubview(slider)
        } else {
            slider.removeFromSuperview()
        }
}
    
func showItem(_ index: Int) {
        print("tapped item \(index)")
        let imageView = UIImageView(image: UIImage(named: "summericons_100px_0\(index).png"))
        imageView.backgroundColor = UIColor(red: 0.0, green: 0.0, blue: 0.0, alpha: 0.5)
        imageView.layer.cornerRadius = 5.0
        imageView.layer.masksToBounds = true
        //如果用代码添加的view是想用autolayout，就设置为false,默认是true不使用autolayout
        imageView.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(imageView)
        //为view创建约束
        let conX = imageView.centerXAnchor.constraint(equalTo: view.centerXAnchor)
        let conBottom = imageView.bottomAnchor.constraint(equalTo: view.bottomAnchor, constant: imageView.frame.height)
        let conWidth = imageView.widthAnchor.constraint(equalTo: view.widthAnchor, multiplier: 0.33, constant: -50.0)
        let conHeight = imageView.heightAnchor.constraint(equalTo: imageView.widthAnchor)
        //相当于设置active = true
        NSLayoutConstraint.activate([conX, conBottom, conWidth, conHeight])
        view.layoutIfNeeded()
        
        UIView.animate(withDuration: 0.8, delay: 0.0, usingSpringWithDamping: 0.4,
                       initialSpringVelocity: 0.0,
                       animations: {
                        conBottom.constant = -imageView.frame.size.height/2
                        conWidth.constant = 0.0
                        self.view.layoutIfNeeded()
        },
                       completion: nil
        )
        
        UIView.animate(withDuration: 0.8, delay: 1.0, usingSpringWithDamping: 0.4,
                       initialSpringVelocity: 0.0,
                       animations: {
                        conBottom.constant = imageView.frame.size.height
                        conWidth.constant = -50.0
                        self.view.layoutIfNeeded()
        },
                       completion: {_ in
                        imageView.removeFromSuperview()
        })
}
```
### CALayer动画
1. 一个动画可被多个layer服用
2. 可设置delegate
3. 可设置key
```swift
        let flyRight = CABasicAnimation(keyPath: "position.x")
        flyRight.fromValue = -view.bounds.size.width/2
        flyRight.toValue = view.bounds.size.width
        flyRight.duration = 0.5
        //fillMode属性值（要想fillMode有效，最好设置removedOnCompletion = false）
        
        //kCAFillModeRemoved 这个是默认值，也就是说当动画开始前和动画结束后，动画对layer都没有影响，动画结束后，layer会恢复到之前的状态
        //kCAFillModeForwards 当动画结束后，layer会一直保持着动画最后的状态
        //kCAFillModeBackwards 在动画开始前，只需要将动画加入了一个layer，layer便立即进入动画的初始状态并等待动画开始。
        //kCAFillModeBoth 这个其实就是上面两个的合成，动画加入之后在开始之前，layer便处于动画初始状态，动画结束后layer保持动画最后的状态
        //默认true。动画结束后，是否移除动画导致的最终效果。如果是false的话，这不移除最终效果，则动画结束后是什么样就是什么样
        flyRight.isRemovedOnCompletion = false
        flyRight.fillMode = kCAFillModeBackwards

        flyRight.delegate = self
        flyRight.setValue("form", forKey: "name")
        flyRight.setValue(heading.layer, forKey: "layer")

        heading.layer.add(flyRight, forKey: nil)
        
        flyRight.beginTime = CACurrentMediaTime() + 0.1
        username.layer.add(flyRight, forKey: nil)
        
        flyRight.beginTime = CACurrentMediaTime() + 0.3
        password.layer.add(flyRight, forKey: nil)

extension ViewController: CAAnimationDelegate {
    func animationDidStop(_ anim: CAAnimation, finished flag: Bool) {
        print("animation did finish")
        
        guard let name = anim.value(forKey: "name") as? String else {
            return
        }
        
        if name == "form" {
            //form field found
            if let layer = anim.value(forKey: "layer") as? CALayer{
                anim.setValue(nil, forKey: "layer")
                
                let pulse = CABasicAnimation(keyPath: "transform.scale")
                pulse.fromValue = 1.25
                pulse.toValue = 1.0
                pulse.duration = 0.25
                layer.add(pulse, forKey: nil)
            }
        }
    }
}
```
### 动画组
```swift
        let groupAnimation = CAAnimationGroup()
        groupAnimation.beginTime = CACurrentMediaTime() + 0.5
        groupAnimation.duration = 0.5
        groupAnimation.fillMode = kCAFillModeBackwards
        groupAnimation.timingFunction = CAMediaTimingFunction(name: kCAMediaTimingFunctionEaseOut)
        
        let scaleDown = CABasicAnimation(keyPath: "transform.scale")
        scaleDown.fromValue = 3.5
        scaleDown.toValue = 1
        
        let rotate = CABasicAnimation(keyPath: "transform.rotation")
        rotate.fromValue = M_PI_4
        rotate.toValue = 0
        
        let fade = CABasicAnimation(keyPath: "opacity")
        fade.fromValue = 0
        fade.toValue = 1
        
        groupAnimation.animations = [scaleDown,rotate,fade]
        loginButton.layer.add(groupAnimation, forKey: nil)
```
```swift
        //动画重复次数,设置0.5表示只执行第一次动画效果，不执行返回动画
        flyLeft.repeatCount = 2.5
        //动画执行完后原路返回
        flyLeft.autoreverses = true
        //动画的速度*2
        flyLeft.speed = 2
        //控件info的速度*2
        info.layer.speed = 2
        //整个view的所有控件速度*2
        view.layer.speed = 2
```
