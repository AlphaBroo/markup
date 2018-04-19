# 时间复杂度

## 算法特性

```
输入：算法具有0个或多个输入
输出：算法至少有1个或多个输出
有穷性：算法在有限的步骤之后会自动结束而不会无线循环，并且每一个步骤可以在可接受的时间内完成
确定性：算法中的每一步都有确定的含义，不会出现二义性
可行性：算法的每一步都是可行的，也就是说每一步都能够执行有限的次数完成
```

## 计算规则

- 基本操作

```
只有常数项,就执行1次

时间复杂度：O(1)
```

- 顺序结构

```
时间复杂度按加法进行计算，一步一步地执行下去

O(n)
```

- 循环结构

```
时间复杂度按乘法计算

简单循环：O(n**3)
递归循环：O(logn)
```

- 分支结构

```
时间复杂度取最大值

多分支if语句，找一个时间最长的作为标准时间
```

- 其他

```
没有特殊说明情况下，所分析的算法的时间复杂度都是指最坏时间复杂度
```

## 常见时间复杂度

| 执行次数函数       | 阶      | 非正式术语 |
| ------------------ | ------- | ---------- |
| 12                 | O(1)    | 常数阶     |
| `2n+3`             | O(n)    | 线性阶     |
| `3n**2+2n+1`       | O(n**2) | 平方阶     |
| `5log2**n+20`      | O(logn) | 对数阶     |
| `6n**3+2n**2+3n+4` | O(n**3) | 立方阶     |

## 性能测试

timeit模块可以测试python代码执行速度

```python
class timeit.Timer(stmt="pass", setup="pass", timer=<timer function>)
# 参数
Timer是测量小段代码执行速度的类
stmt是要测试的代码语句,eg:"func()"
setup是运行代码时需要带的设置，eg:"from __main__ import func"
timer是一个定时器函数，与操作系统平台有福安，一般省略
eg:
    timer1 = timeit.Timer("T1()", "from __main__ import T1")

    
timer.Timer.timeit(number=10000)
# 参数
Timer类中测试语句执行速度的对象方法
number是测试代码时的测试此处，默认1000000次
timeit方法返回执行代码的平均耗时，是float类型的秒数
eg:
    timer1.timeit(1000)
```
