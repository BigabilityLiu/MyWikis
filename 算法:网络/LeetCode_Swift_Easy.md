### [1. 两数之和](https://leetcode-cn.com/problems/two-sum)
#### 重点
使用字典来提升时间效率
#### 题目描述
给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。
你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

示例:
给定 nums = [2, 7, 11, 15], target = 9
因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
#### 答案
```swift
func twoSum(_ nums: [Int], _ target: Int) -> [Int] {
    var mIndex = 0;
    var dic: [Int:Int] = [:]
    for m in nums{
        let value: Int? = dic[target-m]
        if value != nil && value! != mIndex{
            return [value!, mIndex]
        }
        dic[m] = mIndex;
        mIndex += 1
    }
    return [];
}
```

### [7. 整数反转](https://leetcode-cn.com/problems/reverse-integer)
#### 重点
假设我们的环境只能存储得下 32 位的有符号整数，则其数值范围为 [−2^31,  2^31 − 1]。请根据这个假设，如果反转后整数溢出那么就返回 0。
#### 题目描述
给出一个 32 位的有符号整数，你需要将这个整数中每位上的数字进行反转。
示例 1:
输入: 123
输出: 321
示例 2:
输入: -123
输出: -321
示例 3:
输入: 120
输出: 21
#### 答案
```swift
func reverse(_ x: Int) -> Int {
    var x = x
    var result: Int = 0
    while x != 0 {
        let m = x % 10
        x = (x - m) / 10
        if (result > Int32.max/10 || (result == Int32.max/10 && m >  Int32.max%10)) {
            return 0
        }
        if (result < Int32.min/10 || (result == Int32.min/10 && m <  Int32.min%10)) {
            return 0
        }
        result = result*10 + m
    }
    return result
}
```
### [70. 爬楼梯](https://leetcode-cn.com/problems/climbing-stairs/)
#### 重点
动态规划
#### 题目描述
假设你正在爬楼梯。需要 n 阶你才能到达楼顶。
每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？
注意：给定 n 是一个正整数。

示例 1：
输入： 2
输出： 2
解释： 有两种方法可以爬到楼顶。
1.  1 阶 + 1 阶
2.  2 阶
示例 2：

输入： 3
输出： 3
解释： 有三种方法可以爬到楼顶。
1.  1 阶 + 1 阶 + 1 阶
2.  1 阶 + 2 阶
3.  2 阶 + 1 阶
#### 答案
```swift
    func climbStairs(_ n: Int) -> Int {
        var array = Array.init(repeating: 0, count: n + 1)
        array[0] = 0
        array[1] = 1
        array[2] = 2
        if n > 2{
            for i in 3...n {
                array[i] = array[i - 1] + array[i - 2]
            }
        }
        return array[n]
    }
```




=== 模版 ===
### []()
#### 重点

#### 题目描述

#### 答案
```swift

```
