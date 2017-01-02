> There should be one-- and preferably only one --obvious way to do it.<br>
> Although that way may not be obvious at first unless you're Dutch.<br>
> ——[the Zen of Python](https://www.python.org/dev/peps/pep-0020/)

Python提倡用一种，而且最好是只有一种方法来完成一件事。可是在字符串格式化方面Python好像没有做到这一点。除了基本的`%-formatting`格式，还有`str.format()`格式、以及`string.Template`。根据对Python标准库的统计，目前`string.Template`的使用屈指可数，`str.format()`获得广泛使用，但是我们来与其他几种语言的字符串格式化对比一下：

已知`name = 'Tom'`，我们如何打印出字符串`'My name is Tom.'`？

Ruby:

    :::ruby
    puts 'My name is #{name}.'

JavaScript(ECMAScript 2015):

    :::JavaScript
    console.log(`My name is ${name}.`)

Python:

    :::Python
    print('My name is {name}.'.format(name = name))

    # 即便是简化的版本
    print('My name is {}.'.format(name))

可以看出Python明显还不够简洁，于是，随着Python3.6版本在上周正式发布，Python**又**提供了一种字符串格式化语法——'f-strings'。

## f-strings

要使用f-strings，只需在字符串前加上`f`，语法格式如下：

    f ' <text> { <expression> <optional !s, !r, or !a> <optional : format specifier> } <text> ... '

### 基本用法

    :::Python
    >>> name = "Tom"
    >>> age = 3
    >>> f"His name is {name}, he's {age} years old."
    >>> "His name is Tom, he's 3 years old."

### 支持表达式

    :::Python
    # 数学运算
    >>> f'He will be { age+1 } years old next year.'
    >>> 'He will be 4 years old next year.'

    # 对象操作
    >>> spurs = {"Guard": "Parker", "Forward": "Duncan"}
    >>> f"The {len(spurs)} players are: {spurs['Guard']} the guard, and {spurs['Forward']} the forward."
    >>> 'The 2 players are: Parker the guard, and Duncan the forward.'

    >>> f'Numbers from 1-10 are {[_ for _ in range(1, 11)]}'
    >>> 'Numbers from 1-10 are [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]'

### 排版格式

    :::Python
    >>> def show_players():
        print(f"{'Position':^10}{'Name':^10}")
        for player in spurs:
            print(f"{player:^10}{spurs[player]:^10}")
    >>> show_players()
     Position    Name   
      Guard     Parker  
     Forward    Duncan 

### 数字操作

    :::python
    # 小数精度
    >>> PI = 3.141592653
    >>> f"Pi is {PI:.2f}"
    >>> 'Pi is 3.14'

    # 进制转换
    >>> f'int: 31, hex: {31:x}, oct: {31:o}'
    'int: 31, hex: 1f, oct: 37'

### 与原始字符串联合使用

    >>> fr'hello\nworld'
    'hello\\nworld'

## 注意事项

### `{}`内不能包含反斜杠`\`

    :::python
    f'His name is {\'Tom\'}'
    SyntaxError: f-string expression part cannot include a backslash

    # 而应该使用不同的引号，或使用三引号。
    >>> f"His name is {'Tom'}"
    'His name is Tom'

### 不能与`'u'`联合使用

`'u'`是为了与Python2.7兼容的，而Python2.7不会支持f-strings，因此与`'u'`联合使用不会有任何效果。

### 如何插入大括号？

    :::python
    >>> f"{{ {10 * 8} }}"
    '{ 80 }'
    >>> f"{{ 10 * 8 }}"
    '{ 10 * 8 }'

### 与`str.format()`的一点不同

使用`str.format()`，非数字索引将自动转化为字符串，而f-strings则不会。

    :::python
    >>> "Guard is {spurs[Guard]}".format(spurs=spurs)
    'Guard is Parker'

    >>> f"Guard is {spurs[Guard]}"
    Traceback (most recent call last):
      File "<pyshell#34>", line 1, in <module>
        f"Guard is {spurs[Guard]}"
    NameError: name 'Guard' is not defined

    >>> f"Guard is {spurs['Guard']}"
    'Guard is Parker'

---
参考：

+ [PEP 498 -- Literal String Interpolation](https://www.python.org/dev/peps/pep-0498/#similar-support-in-other-languages)

+ [printf-style String Formatting](https://docs.python.org/3/library/stdtypes.html#printf-style-string-formatting)

+ [Format String Syntax](https://docs.python.org/3/library/string.html#formatstrings)