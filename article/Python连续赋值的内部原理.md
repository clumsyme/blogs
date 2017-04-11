## C中的连续赋值

在 C 中，连续赋值执行顺序是从右到左，如下边语句：

    :::C
    a = b = c = 3

其实相当于

    :::C
    a = (b = (c = 3))

首先执行`c = 3`，该表达式的值为 3，然后执行`b = (c = 3)`；b赋值为 3，该表达式值为 3；最后执行`a = (b = (c = 3))`，a 赋值为 3。

## Python中的连续赋值问题

长期以来我都将 Python 中的连续赋值以上述方法对待，但其实并不是这样的。看下边这段代码：

    :::Python
    foo = [0]
    bar = foo
    foo[0] = foo = [1]

    print(foo)
    print(bar)

如果按 C 中的连续赋值处理方式进行处理，那么在连续赋值语句中将：

+ 首先执行`foo = [1]`，结果就是`foo`指向了一个新建的对象，而`bar`是`foo`原对象的一个引用，`bar`值将保持原值（[0]）

+ 然后执行`foo[0] = (foo = [1])`也即`foo[0] = [1]`

结果应该是输出

    :::Python
    [[1]]
    [0]

而实际输出确是

    :::Python
    [1]
    [[1]]

## Python连续赋值研究

那么到底是怎么回事呢，借助`dis`模块我们来一探究竟：

    :::Python
    >>> import dis
    >>> dis.dis('foo[0] = foo = [1]')
      1         0 LOAD_CONST               0 (1)
                3 BUILD_LIST               1
                6 DUP_TOP
                7 LOAD_NAME                0 (foo)
               10 LOAD_CONST               1 (0)
               13 STORE_SUBSCR
               14 STORE_NAME               0 (foo)
               17 LOAD_CONST               2 (None)
               20 RETURN_VALUE

注意这几行

+ `6 DUP_TOP`将构建的列表在栈顶复制了一份

+ `13 STORE_SUBSCR`与`14 STORE_NAME   0 (foo)`: python首先执行的是将栈顶的 [1] 赋值给 foo[1]，而 foo 正是原对象，然后才将另一个 [1] 赋值给 foo, 结果就是原对象（同时也是 bar 指向的对象）变成了 [[1]] 然后 foo 被赋予新值 [1]。

所以 Python 中的连续赋值其实是下边的顺序：

+ 首先构建要赋值的对象

+ 将对象在栈顶进行一份复制，然后将复制的值赋给第一个变量

+ 将对象在栈顶进行一份复制，然后将复制的值赋给第二个变量

+ ……

+ 将对象赋值给最后一个变量

整个赋值是从左向右的。

## ps：交换变量赋值

顺便看一下交换变量赋值的原理

    :::Python
    >>> dis.dis('a, b = b, a')
      1           0 LOAD_NAME                0 (b)
                  3 LOAD_NAME                1 (a)
                  6 ROT_TWO
                  7 STORE_NAME               1 (a)
                 10 STORE_NAME               0 (b)
                 13 LOAD_CONST               0 (None)
                 16 RETURN_VALUE

+ 首先将变量 b 的值推入栈

+ 然后将变量 a 的值推入栈

+ 交换栈顶的两元素（结果是 b 的值位于栈顶）

+ 栈顶的值（b）赋值给变量 a

+ 栈顶的值（a）赋值给变量 b