我们都知道在函数体内使用`yield`关键字将会使函数返回一个生成器对象，在英文字典中，yield不仅有“生产”(produce)的意思，还有“屈服，让步”(give way)的意思，因此python最开始也使用了生成器的关键字`yield`来处理**协程(coroutine)**。但是不同的是，协程中`yield`关键字通常出现于表达式的右侧，例如`received = yield value`；协程不仅可以生成数据（或者不生成数据），也可以接收数据。理解协程中的`yield`，要把`yield`看做是用来执行控制流的，而不是生成数据的。

## python协程的历史

python协程最早的实现是在python2.5中，自此可以在表达式中使用`yield`关键字了，同时为生成器实现了.send(value)方法用于向`yield`表达式传递数据，.throw()方法用于抛出异常，.close()方法用于终结。具体描述在[PEP 342](https://www.python.org/dev/peps/pep-0342/)中。

之后由于嵌套生成器的缘故，在python3.3中实现对协程进行了改进（见[PEP 380](https://www.python.org/dev/peps/pep-0380/)）：

+ 生成器现在可以返回值了，之前在生成器内返回值会触发SyntaxError

+ 新的`yield from`语法使得复杂的、嵌套的生成器代码更加简洁。

使用同一个关键字很容易让人迷惑，因此在python3.5，python借鉴了其他语言，添加了两个新的关键字`async`和`await`用来处理协程，使得协程代码更容易让人理解。

## 初步了解协程

    >>> def simple_coroutine():
        print("What's your name?")
        name = yield
        print("Hello", name, ", where are you from?")
        nation = yield
        print("OK, I see")

    ##############################
    >>> coro = simple_coroutine()
    >>> coro
    <generator object simple_coroutine at 0x01156480>       #1
    >>> next(coro)                                          #2
    What's your name?
    >>> coro.send("Li Yan")                                 #3
    Hello Li Yan , where are you from?
    >>> coro.send("China")
    OK, I see
    Traceback (most recent call last):                      #4
      File "<pyshell#79>", line 1, in <module>
        coro.send("China")
    StopIteration


`#1`显示，调用函数返回的是一个生成器对象。`#2`使用next()方法是因为生成器此时还未启动，它并未在一个`yield`等待因此我们还不能向它传递数据，调用next()后生成器就会启动。`#3`我们调用生成器的`.send()`方法，使得协程内`yield`等于5，同时协程继续运行，直到再次遇到`yield`或者自身终止。`#4`处协程(生成器)运行到了最后，就如生成器一般表现一样，触发了`StopIteration`错误。

## 协程的四个状态

每个协程可能处于四个状态，我们来观察一下：

    >>> from inspect import getgeneratorstate
    >>> coro = simple_coroutine()
    >>> getgeneratorstate(coro)
    'GEN_CREATED'
    >>> next(coro)
    What's your name?
    >>> getgeneratorstate(coro)
    'GEN_SUSPENDED'
    >>> coro.send('Li Yan')
    Hello Li Yan , where are you from?
    >>> getgeneratorstate(coro)
    'GEN_SUSPENDED'
    >>> coro.send('China')
    OK, I see
    Traceback (most recent call last):
      File "<pyshell#92>", line 1, in <module>
        coro.send('China')
    StopIteration
    >>> getgeneratorstate(coro)
    'GEN_CLOSED'

除了'GEN_CREATED'，'GEN_SUSPENDED'，'GEN_CLOSED'，还有'GEN_RUNNING'，不过我们很少观察到此状态。

上述整个过程运行流程如下：

通过`coro = simple_coroutine()`我们得到了一个协程（生成器），此时它处于'GEN_CREATED'状态，我们还不能向它传递值，通过调用next(coro)启动协程，直到运行到`yield`暂停，等待我们传入数据，`coro.send('Li Yan')`使`yield`赋值为'Li Yan'，`name`获得值'Li Yan'，并继续运行直到下一个`yield`，在我们传入数据'China'之后，`nation`获得值'China'并继续运行，直到触发了`StopIteration`，此时协程处于'GEN_CLOSED'状态。

## 使用协程实现平均数函数

在[python中的闭包与局部作用域](http://www.imliyan.com/blogs/article/python中的闭包与局部作用域/)一文中，我们使用闭包实现了一个平均数函数，每次接收一个参数，返回之前每次调用传给它的所有参数的平均值。这里我们用协程来实现

    def averager():
        total = 0
        count = 0
        average = None
        while True:
            new = yield average
            total += new
            count += 1
            average = total / count
    
    ####################
    >>> avg = averager()
    >>> next(avg)
    >>> avg.send(10)
    10.0
    >>> avg.send(11)
    10.5

使用协程，我们不必使用闭包，也不必声明非局部变量(nonlocal)。但是这个协程是个无限循环，所以我们应该如何停止该协程呢？

## 协程终止与异常处理

要终止一个协程，最直接的办法是传递一个错误的参数（例如向avg传递一个字符串），协程内部触发错误而未得到处理，协程就将终止。

    >>> avg.send('foo')
    Traceback (most recent call last):
      File "<pyshell#110>", line 1, in <module>
        avg.send('foo')
      File "<pyshell#103>", line 7, in averager
        total += new
    TypeError: unsupported operand type(s) for +=: 'int' and 'str'
    >>> getgeneratorstate(avg)
    'GEN_CLOSED'

自python2.5引入`.throw()`和`.close()`方法后，可一更方便地控制协程了，

    def averager():
        total = 0
        count = 0
        average = None
        while True:
            new = yield average
            try:
                total += new
                count += 1
            except TypeError:
                print('Wrong value')
            else:
                average = total / count
    
    #####################
    >>> avg = averager()
    >>> next(avg)
    >>> avg.send(10)
    10.0
    >>> avg.send('foo')
    Wrong value
    10.0
    >>> avg.send(11)
    10.5
    >>> avg.close()
    >>> getgeneratorstate(avg)
    'GEN_CLOSED'

## 使协程返回值

对于上述平均数函数，我们可能并不关心中间每次计算的平均数，而只关心最终的平均数，自python3.3后，协程可以返回特定值了。

    def averager2():
        total = 0
        count = 0
        average = None
        while True:
            new = yield
            if new == None:
                break
            total += new
            count += 1
            average = total / count
        return {'count': count, 'average': average}
    
    #########################
    >>> avg2 = averager2()
    >>> next(avg2)
    >>> avg2.send(10)
    >>> avg2.send(11)
    >>> avg2.send(12)
    >>> avg2.send(None)
    Traceback (most recent call last):
      File "<pyshell#143>", line 1, in <module>
        avg2.send(None)
    StopIteration: {'count': 3, 'average': 11.0}

我们使用`avg2.send(None)`终止协程，并希望返回一个携带数据的字典，但我们看到数据最终是作为`StopIteration`错误的值返回的。那么我们如何获得返回的值呢？

    >>> try:
        avg2.send(None)
    except StopIteration as exc:
        data = exc.value

    >>> data
    {'count': 3, 'average': 11.0}

为了获得协程返回的数据，这是一个迂回的办法，幸好[PEP 380](https://www.python.org/dev/peps/pep-0380/)的`yield from`可以自动帮我们处理这种情况：使用`yield from`，解释器不仅自动处理了`StopIteration`，并且是`StopIteration`的`value`属性成为`yield from`表达式的值。这就像是使用`for`来处理可迭代对象(Iterable)一样。

## 使用 yield from

### 生成器中的 yield from

如果只是用于产生值，yield from 可以更直观地替代 for 循环中的 yield，例如：

    :::python
    def gen():
        for c in 'ABCDE':
            yield c

可以写作:

    :::python
    def gen():
        yield from 'ABCDE'

如果 yield from 只是作为一个语法糖类似的功能的话，python 可能也不会接受它作为新的语言特性。它真正的作用是delegating generator ：最外层的 caller(PEP380 使用的术语，指调用 delegating generator 的对象) 传递数据给 delegating generator ，实际上通过 delegating generator 传递给了subgenerator，而 subgenerator 生成的数据反过来通过 delegating generator 传递回 caller。

如果 subgenerator 有返回值，则 delegating generator 会自动处理`StopIteration`，并将返回值赋予 yield from 表达式。

    :::python
    # 作为 subgenerator 
    def averager():
        '''上一节介绍的平均协程程序，此处为 subgenerator '''
        total = 0
        count = 0
        average = None
        while True:
            # caller 传递的数据将传递到这里
            new = yield
            # 用于终结 while 循环
            if new == None:
                break
            total += new
            count += 1
            average = total / count
        # 生成器返回值
        return {'count': count, 'average': average}

    #  delegating generator 
    def grouper(results, key):
        ''' delegating generator '''
        while True:
            # 所有从 caller 传递的数据都通过 yield from 传递给 subgenerator 
            # 直到 subgenerator 停止并返回，返回值将赋值给result[key]
            results[key] = yield from averager()

    # caller
    def main(data):
        '''客户端代码，也就是 caller'''
        results = {}
        for key, values in data.items():
            # group 是一个生成器对象，可以像协程一样操作它
            group = grouper(results, key)
            # 启动协程
            next(group)
            for value in values:
                # 传递数据，数据将传递到 new = yield, grouper 不会接触到改数据
                group.send(value)
            # 结束 subgenerator ，使 delegating generator 继续运行
            group.send(None)
        report(results)
    
    def report(results):
        for key, result in results.items():
            group, unit = key.split(';')
            print('{:2} {:5} averaging {:.2f}{}'.format(
                result['count'], group, result['average'], unit
            ))

    data = {
        'girls;kg':
            [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5],
        'girls;m':
            [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43],
        'boys;kg':
            [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3],
        'boys;m':
            [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
    }
    
    # 运行上述代码
    >>> main(data)
     9 boys  averaging 1.39m
    10 girls averaging 42.04kg
     9 boys  averaging 40.42kg
    10 girls averaging 1.43m

描述一下整个 main 函数运行流程：

+ 每个外层 for 循环生成一个新的 grouper 实例，也就是 delegating generator 
+ next(group) 启动 delegating generator ，并在调用 subgenerator 之后在 yield from 处暂停
+ 内层 for 循环直接将数据传递给 subgenerator ，与此同时 delegating generator 仍在 yield from 处暂停
+ 当内层 for 循环结束时，group仍然在 yield from 处暂停，所以results[key]赋值还未完成
+ 传递 None 给 delegating generator （实际上传递给了 subgenerator ）以终结 subgenerator ，并使 delegating generator 继续运行，完成赋值
+ 外层 for 循环继续运行，生成新的 grouper 实例，上一个 grouper 被垃圾回收