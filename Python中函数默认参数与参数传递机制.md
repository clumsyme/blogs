在[Python中的闭包与局部作用域](http://www.imliyan.com/blogs/article/Python中的闭包与局部作用域/)一文最后，我们用函数默认参数代替闭包，实现了[记忆化](https://zh.wikipedia.org/wiki/%E8%AE%B0%E5%BF%86%E5%8C%96)。这篇文章就更深入的研究一下Python中的函数默认参数。

## 关于默认参数

下列代码的运行结果可能会让很多刚接触Python的同学感到意外

    :::Python
    def foo(li=[]):
        li.append(1)
        return li

    ###############
    >>> foo()
    [1]
    >>> foo([5])
    [5, 1]
    >>> foo()
    [1, 1]
    >>> foo()
    [1, 1, 1]

通常我们认为`foo()`应该返回`[1]`，是因为我们认为在每次函数调用时`li`都是`[]`，而事实上不是。[stackoverflow](http://stackoverflow.com/questions/1132941/least-astonishment-and-the-mutable-default-argument)上有一个很好的解释————作为第一类对象的函数对象，在其首次定义时默认参数就作为函数对象的“成员数据”而存在了，后续的状态修改自然会影响其值。而在[Fluent Python](http://shop.oreilly.com/product/0636920032519.do)一书中对此也有很清晰的解释（十分推荐这本书）：

> The problem is that each default value is evaluated when the function is defined — i.e. usually when the module is loaded — and the default values become attributes of the function object. So if a default value is a mutable object, and you change it, the change will affect every future call of the function.

我们一旦定义了函数，默认参数值就成了函数对象属性的一部分，对于可变默认参数值的修改将会影响后续的函数调用。我们通过两个函数来验证这一点：

### 不可变默认参数

    :::Python
    def unmutable(var=1):
        print(id(var), var)
        var += 1
        print(id(var), var)

    ####################
    >>> unmutable()
    1626866128 1       #1
    1626866160 2       #2
    >>> unmutable(2)
    1626866160 2       #3
    1626866192 3       #4
    >>> unmutable()
    1626866128 1       #5
    1626866160 2       #6

+ `#1` 与 `#2` 对比可知，对不可变对象进行修改后，`var`其实已经是另外一个对象了
+ `#3`显示`var`不是默认对象。
+ `#5` 显示再次调用`var`仍然是原对象。

### 可变默认参数

    :::Python
    def mutable(var=[]):
        print(id(var), var)
        var.append(1)
        print(id(var), var)

    ######################
    >>> mutable()
    2326890888520 []       #1
    2326890888520 [1]      #2
    >>> mutable([2])
    2326898722504 [2]      #3
    2326898722504 [2, 1]   #4
    >>> mutable()
    2326890888520 [1]      #5
    2326890888520 [1, 1]   #6

+ `#1` 与 `#2` 对比可知，对可变对象进行修改后，`var`仍为原对象。
+ `#3`显示`var`不是默认对象。
+ `#5` 显示再次调用`var`仍为原对象，但其值已经改变。


再看下边的代码

    :::Python
    def mutorno(var=[]):
        print(id(var), var)
        var = var+[1]
        print(id(var), var)

    ######################
    >>> mutorno()
    2326987935304 []        #1
    2326987885384 [1]       #2

+ 对`var`进行赋值后，`var`已经不再是原对象了。

至此我们可以总结：如果默认参数为不可变对象，在函数体内对该参数的修改都将导致参数名指向另外一个对象，而原对象的值不变；若为可变对象，对其进行的修改将直接影响原对象的值，而对其重新赋值将导致参数名指向另一个对象。

## Python的参数传递机制

    :::Python
    i = 1
    def f(obj):
        obj += 1
    >>> f(i)
    >>> i
    1
    
    li = []
    def g(obj):
        obj += [1]
    >>> g(li)
    >>> li
    [1]

如果Python是按引用传递，那`f`函数调用后`i`的值应该为2；如果Python是按值传递，那`g`函数调用后`li`的值应该为`[]`。那么Python的参数传递机制到底是什么呢？

Fluent Python一书中将其描述为共享传递(call by sharing)

如果Python是按引用传递，那`f`函数调用后`i`的值应该为2；如果Python是按值传递，那`g`函数调用后`li`的值应该为`[]`。那么Python的参数传递机制到底是什么呢？

Fluent Python一书中将其描述为共享传递(call by sharing)，

> The only mode of parameter passing in Python is call by sharing. That is the same mode used in most OO languages, including Ruby, SmallTalk and Java. Call by sharing means that each formal parameter of the function gets a copy of each reference in the arguments. In other words, the parameters inside the function become aliases of the actual arguments.<br>
> The result of this scheme is that a function may change any mutable object passed as a parameter, but it cannot change the identity of those objects, i.e. it cannot replace altogether an object with another. <br>
———— fluent Python

也就是在f函数中，obj 和 i 都是对同一个对象的引用，g 函数中，li 和 obj 都是对同一个对象的引用。而由于 int 类型为不可变对象，在函数体内对其修改将使其变为另一个对象的引用，修改的是另一个对象，因此原对象是不变的。而 list 类型为可变的，`+=` 是 in place 操作，直接修改了原对象的值。

这一点和JavaScript的基本对象/引用对象传递参数机制类似，在传递基本对象时表现为按值传递，传递引用对象时表现为按引用传递。但JavaScript的参数传递机制只有按值传递，

> ECMAScript中的所有参数传递都是值，不可能通过引用传递参数。<br>
> 基本类型值指的是简单的数据段，而引用类型值指那些可能由多个值构成的对象。<br>
>————JavaScript 高级程序设计 第三版

*[stackoverflow上有一个解释也将JavaScript的参数传递机制解释为call by sharing。](http://stackoverflow.com/questions/518000/is-javascript-a-pass-by-reference-or-pass-by-value-language/3638034#3638034)*