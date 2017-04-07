人们常将闭包与匿名函数混淆，因为在函数体内定义函数并不十分常见，直到我们要用到匿名函数，而只有在嵌套函数的情况下讨论闭包才有意义。

事实上，闭包与匿名函数并无关系。闭包指的是函数包含一个扩展的作用域，该作用域内的非全局变量不是在该函数体内定义，但该函数可以引用这些变量，即使这些变量离开了创建它的环境。它与函数是否匿名无关。
[segmentfault](https://segmentfault.com/q/1010000007578832)上有一个形象的解释：
*闭包是自带运行环境的函数*。

## 作用域相关

在继续讨论闭包之前，先来看一看Python中作用域容易让人疑惑的一些地方。

    :::Python
    def p(a):
        print(a)
        print(b)
    
    ######################
    >>> p(1)
    1
    Traceback (most recent call last):
      File "<pyshell#17>", line 1, in <module>
        p(1)
      File "<pyshell#16>", line 3, in p
        print(b)
    NameError: name 'b' is not defined

很容易理解，`b`并未定义。

    :::Python
    b = 2

    ###############
    >>> p(1)
    1
    2
    
上述代码运行正常。

    :::Python
    b=2
    def p2(a):
        print(a)
        print(b)
        b=3

    ################
    >>> p2(1)
    1
    Traceback (most recent call last):
      File "<pyshell#22>", line 1, in <module>
        p2(1)
      File "<pyshell#19>", line 3, in p2
        print(b)
    UnboundLocalError: local variable 'b' referenced before assignment

上述代码`print(b)`是在局部变量b定义之前调用的，我们很容易认为`print(b)`中的`b`是全局变量`b`,但事实却不是如此。在Python中，万物皆对象，`function`也是`first-class object`，这也就意味着在函数p2定义时，就决定了`b`是局部变量，在调用`print(b)`时，将会尝试获取局部变量`b`，而此时局部变量`b`还未定义，因此出错。

我们来用`dis`模块查看函数调用过程中都发生了什么。

    :::Python
    from dis import dis
    >>> dis(p)
      2           0 LOAD_GLOBAL              0 (print)
                  3 LOAD_FAST                0 (a)
                  6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
                  9 POP_TOP
    
      3          10 LOAD_GLOBAL              0 (print)
                 13 LOAD_GLOBAL              1 (b)
                 16 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
                 19 POP_TOP
                 20 LOAD_CONST               0 (None)
                 23 RETURN_VALUE

    >>> dis(p2)
      2           0 LOAD_GLOBAL              0 (print)
                  3 LOAD_FAST                0 (a)
                  6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
                  9 POP_TOP
    
      3          10 LOAD_GLOBAL              0 (print)
                 13 LOAD_FAST                1 (b)
                 16 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
                 19 POP_TOP
    
      4          20 LOAD_CONST               1 (3)
                 23 STORE_FAST               1 (b)
                 26 LOAD_CONST               0 (None)
                 29 RETURN_VALUE

注意这两行：

+ `13 LOAD_GLOBAL              1 (b)`
+ `13 LOAD_FAST                1 (b)`

可见在函数p中，print(b)使用全局变量，而函数p2中，使用的是局部变量。

## 闭包

假如我们需要定义一个函数，该函数每次接收一个参数，返回之前每次调用传给它的所有参数的平均值。我们首先尝试使用可调用类实例来实现。

    :::Python
    class Averager:
        def __init__(self):
            self.values = []
        
        def __call__(self, new_value):
            self.values.append(new_value)
            return sum(self.values) / len(self.values)

    #####################
    >>> avg = Averager()
    >>> avg(10)
    10
    >>> avg(11)
    10.5

我们使用了实例属性来存储所有传递进来的参数值，下边我们将使用函数来实现。

    :::Python
    def get_avg():
        values = []
        def avg(value):
            values.append(value)
            return sum(values) / len(values)
        return avg
    
    #####################
    >>> avg = get_avg()
    >>> avg(10)
    10
    >>> avg(11)
    10.5

当我们调用`avg(value)`时，`get_avg`的作用域已经不再存在了，但我们仍然可以获得values的值。values在avg的作用域内作为自由变量(free variable)存在，自由变量意味着values并未绑定avg的作用域。我们来研究avg函数/对象：

    :::Python
    >>> avg.__code__.co_varnames
    ('value',)
    >>> avg.__code__.co_freevars
    ('values',)
    >>> avg.__closure__
    (<cell at 0x000001BAFF2E1AC8: list object at 0x000001BAFF3174C8>,)
    >>> avg.__closure__[0].cell_contents
    [10,11]

如此我们就实现了闭包，使变量在其作用域不再存在的情况下依然存在。**函数保持与其定义时的自由变量的绑定，并且可以在之后继续使用**。

## 还是作用域

注意到上述函数实现在每次计算时都要计算`values`的所有值的和，我们可能想要优化一下。

    :::Python
    def get_avg():
        count = 0
        total = 0
        def avg(value):
            count += 1
            total += value
            return total / count
        return avg

    ####################
    >>> avg = get_avg()
    >>> avg(10)
    Traceback (most recent call last):
      File "<pyshell#33>", line 1, in <module>
        avg(10)
      File "<pyshell#31>", line 5, in avg
        count += 1
    UnboundLocalError: local variable 'count' referenced before assignment

可是却又出错了？提示说明得很清楚：在局部变量赋值之前对其进行了引用。这是因为对于 int 类型变量，`count += 1` 等于 `count = count + 1`, `count`成为了avg的局部变量。而在之前我们使用的列表，并未进行赋值操作，因此`values`是自由变量。

解决这个问题，只要声明 count 和 total 为非局部变量即可。

    :::Python
    def get_avg():
        count = 0
        total = 0
        def avg(value):
            nonlocal count, total
            count += 1
            total += value
            return total / count
        return avg

## 不用闭包实现avg函数

通常，在Python中使用可变数据作为函数默认参数会被认为是危险的，因为可变默认参数数据存储于function object的`__default__`属性中，一旦对其进行修改，默认值就不再是定义时的值了。但我们可以利用这点实现记忆化。

    :::Python
    def avg(n, values=[]):
        values.append(n)
        return sum(values) / (len(values))

    def avg(n, dic={"total":0, "count":0}):
        dic["count"] += 1
        dic["total"] += n
        return dic["total"] / dic["count"]