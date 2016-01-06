构造器的使用

[官方文档](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Initialization.html#//apple_ref/doc/uid/TP40014097-CH18-ID203)
[翻译文档](http://wiki.jikexueyuan.com/project/swift/chapter2/14_Initialization.html)
## Initializers
通过定义构造器(Initializers)`e.g. init(){}`来实现构造过程，这些构造器可以看做是用来创建特定类型新实例的特殊方法。与 Objective-C 中的构造器不同，Swift 的构造器无需返回值，它们的主要任务是保证新实例在第一次使用前完成正确的初始化。

类的实例也可以通过定义析构器(deinitializer) `e.g. deinit{}`在实例释放之前执行特定的清除工作。

###默认构造器：如果没有自定义构造器，类和结构体会有默认构造器
类的默认构造器有默认值。<br>
```Swift
class ShoppingListItem {
    var name: String?
    var quantity = 1
}
var item = ShoppingListItem()
```
<br>由于ShoppingListItem类中的所有属性都有默认值，且它是没有父类的基类，它将自动获得一个可以为所有属性设置默认值的默认构造器（尽管代码中没有显式为name属性设置默认值，但由于name是可选字符串类型，它将默认设置为nil）。<br>
结构体的默认构造器没有默认值。如果结构体没有提供自定义的构造器，它们将自动获得一个逐一成员构造器，即使结构体的存储型属性没有默认值。例如下面<br>
###值类型的构造器代理
构造器代理的实现规则和形式在值类型和类类型中有所不同。值类型（结构体和枚举类型）不支持继承，所以构造器代理的过程相对简单.<br>
```Swift
struct Size {
    var width = 0.0, height = 0.0
}
let twoByTwo = Size(width: Double, height: Double)
```
<br>
```Swift
struct Size {
    var width = 0.0, height = 0.0
}
struct Point {
    var x = 0.0, y = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    init() {}
    init(origin: Point, size: Size) {
        self.origin = origin
        self.size = size
    }
}
let rect = Rect()//(0,0,0,0)
let rect1 = Rect(center: Point(x: 2, y: 4), size: Size())//(2,4,0,0)
```
###类的继承和构造过程
类里面的所有存储型属性——包括所有继承自父类的属性——都必须在构造过程中设置初始值。<br>
Swift 为类类型提供了两种构造器来确保实例中所有存储型属性都能获得初始值，它们分别是指定构造器和便利构造器。
[指定构造器和便利构造器](http://wiki.jikexueyuan.com/project/swift/chapter2/14_Initialization.html)