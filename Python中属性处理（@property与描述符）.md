当定义一个类时，我们可以通过定义几个特殊方法来处理属性的获取、设置、删除等操作：

+ `__delattr__(self, name)`

    使用`del`来删除属性时将调用此方法

+ `__getattr__(self, name)`

    当一个属性不存在时，将调用此方法返回属性值

+ `__getattribute__(self, name)`

    不管属性存不存在，都使用此方法返回属性值

+ `__setattr__(self, name, value)`

    所有属性设置操作都将调用此方法

可以看出，都是处理所有的属性获取/设置，`__getattribute__`和`__setattr__`命名并不统一，因此实际使用中若需要处理属性获取/设置，通常使用@property或者描述符。

## @property装饰器

我们可以通过定义方法来控制属性的获取与设置：

    :::Python
    class Dog:
        def __init__(self, age):
            self.__age = age

        def age_getter(self):
            return self.__age
        
        def age_setter(self, age):
            if age >= 0:
                self.__age = age
            else:
                raise ValueError('age must be > 0')

但每次获取/设置属性都要通过调用方法实在太麻烦了，因此我们可以将其包装为一个property实例。这样我们就可以通过`obj.attr`格式来处理对象的属性了。

    :::Python
    class Dog:
        def __init__(self, age):
            self.__age = age

        def age_getter(self):
            return self.__age
        
        def age_setter(self, age):
            if age >= 0:
                self.__age = age
            else:
                raise ValueError('age must be > 0')
        
        age = property(age_getter, age_setter)

可以使用@property装饰器简化上述定义

    :::Python
    class Dog:
        def __init__(self, age):
            self.age = age

        @property
        def age(self):
            return self.__age

        @age.setter
        def age(self, age):
            if age >= 0:
                self.__age = age
            else:
                raise ValueError('age must be > 0')
    
    ###########################
    >>> d = Dog(3)
    >>> d.age
    3
    >>> d.age = 5
    >>> d.age
    5
    >>> dog = Dog(-1)
    Traceback (most recent call last):
      File "<pyshell#78>", line 1, in <module>
        dog = Dog(-1)
      File "<pyshell#72>", line 3, in __init__
        self.age = age
      File "<pyshell#72>", line 14, in age
        raise ValueError('age must be > 0')
    ValueError: age must be > 0
    >>> d.age = -1
    Traceback (most recent call last):
      File "<pyshell#79>", line 1, in <module>
        d.age = -1
      File "<pyshell#72>", line 14, in age
        raise ValueError('age must be > 0')
    ValueError: age must be > 0

如果不设置@age.setter，age将成为只读属性。

使用@property可以解决大部分处理属性获取/设置的需求，不过如果我们还需要处理其他属性，那每个属性都要设置@property, 代码重复就有点多，我们可以定义一个函数：

    :::Python
    def makeproperty(name):
        def name_getter(instance):
            return instance.__dict__[name]
        def name_setter(instance, value):
            if value <= 0:
                raise ValueError
            else:
                instance.__dict__[name] = value
        return property(name_getter, name_setter)

    class Dog:
        age = makeproperty("age")
        weight = makeproperty("weight")
        def __init__(self, age, weight):
            self.age = age
            self.weight = weight

    ########################
    >>> d = Dog(3, 10)
    >>> d.age
    3
    >>> d.weight
    10
    >>> d.weight = -7
    Traceback (most recent call last):
      File "<pyshell#105>", line 1, in <module>
        d.weight = -7
      File "<pyshell#99>", line 6, in name_setter
        raise ValueError
    ValueError
    >>> dog = Dog(-1, 9)
    Traceback (most recent call last):
      File "<pyshell#106>", line 1, in <module>
        dog = Dog(-1, 9)
      File "<pyshell#101>", line 5, in __init__
        self.age = age
      File "<pyshell#99>", line 6, in name_setter
        raise ValueError
    ValueError

其实Python专门有一类对象就是用于处理这种情况的：描述符(descriptor)

## 描述符

> Descriptors are a way of reusing the same access logic in multiple attributes. For example, field types in ORMs such as the Django ORM and SQL Alchemy are descriptors, managing the flow of data from the fields in a database record to Python object attributes and vice-versa.<br>
> **A descriptor is a class which implements a protocol consisting of the __get__, __set__ and __delete__ methods**.<br>
———— Fluent Python

property类就是一个实现了`__get__`与`__set__`方法的描述符

    :::Python
    class Descriptor:
        def __init__(self, name):
            self.name = name

        def __set__(self, instance, value):
            if value > 0:
                instance.__dict__[self.name] = value
            else:
                raise ValueError('{} must be > 0'.format(self.name))

    class Dog:
        age = Descriptor('age')
        weight = Descriptor('weight')
        def __init__(self, age, weight):
            self.age = age
            self.weight = weight

    >>> d = Dog(3, 10)

当我们运行`d.age = -1`时，等同于`Dog.age.__set__(d, -1)`，也就是描述符的`__set__`方法处理了Dog实例属性的设置。由于此描述符未实现`__get__`方法，因此属性的获取仍然是默认的从`d.__dict__`中属性获取。

如果同时实现了`__get__`和`__set__`方法，那描述符将处理属性的设置与获取。

    :::Python
    class Descriptor2:
        def __init__(self, name):
            self.name = name

        def __get__(self, instance, owner):
            return 'Oh NO!'

        def __set__(self, instance, value):
            instance.__dict__[self.name] = value

    class Dog:
        age = Descriptor2('age')
        weight = Descriptor('weight')
        def __init__(self, age, weight):
            self.age = age
            self.weight = weight
    
    ######################
    >>> d = Dog(3, 10)
    >>> d.age
    'Oh NO!'
    >>> d.age = 5
    >>> d.age
    'Oh NO!'
    >>> d.__dict__
    {'weight': 10, 'age': 5}

总结一下实现了不同方法的描述符：

+ 只实现了`__get__`方法：`obj.attr`将首先尝试在`obj.__dict__`中获取属性值，如果没有才调用描述符的`__get__`方法。`obj.attr = value`将会直接修改`obj.__dict__`中的属性值。

+ 只实现了`__set__`方法：`obj.attr`将首先尝试在`obj.__dict__`中获取属性值，如果没有才调用描述符的`__get__`方法。`obj.attr = value`将调用`__set__`方法来处理属性设置。

+ 同时实现了`__get__`和`__set__`方法（overriding descriptor）：`obj.attr`将总是首先调用`__get__`方法来获取属性，`obj.attr = value`也总将调用`__set__`方法来处理属性设置。*@property就是一个overriding descriptor*。

### ps

*方法的实质也是一个描述符，因为用户定义函数都包含一个`__get__`方法，因此当函数作为一个类实例的属性时，它就表现为一个描述符。当通过一个类调用函数时，`__get__`方法返回函数自身，而通过实例调用函数时，`__get__`方法返回一个绑定方法对象(bound method object):一个可被调用并绑定了实例作为其首个参数的函数。因此`obj.method`将是一个方法，而`Obj.methosd`为函数*。