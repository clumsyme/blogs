在这篇文章中，我们会尝试使 JavaScript 的类具有 Python 的风格：在方法内明确地传递和使用 self 参数，但是调用时自动传入实例参数。我们将使用描述符和 Proxy 来进行实现，在此过程中来学习属性描述符、Proxy、Reflect 的概念。

## this 问题

`this` 是 JavaScript 中一个很重要有很迷惑人的东西，我们总是要时刻思考函数内的 `this` 到底是谁。随便翻一翻一些开发论坛，像 **JavaScript 中的 this 指的是什么** 是很高频的文章（实际上，我自己也[写过](https://www.imliyan.com/blogs/article/JavaScript%E4%B8%ADthis%E6%8C%87%E7%9A%84%E6%98%AF%E8%B0%81/)）。那么为什么像 Python 就没有这个问题呢。

对于下边这个 Python 类：

```py
class Dog:
    def __init__(self, name):
        self.name = name
    def bark(self):
        print(f'Wang! {self.name}')
tom = Dog('Tom')
tom.bark()
# Wang! Tom

bark = tom.bark
bark()
# Wang! Tom

snoopy = Dog('Snoopy')
snoopy.bark = bark
snoopy.bark()
# Wang! Tom
```

无论怎么调用 `dog.bark`, 函数内的 `self` 永远都是 `tom`。

> Explicit is better than implicit.
>
> ———— The Zen of Python

Python 中的 `self` 是明确传参，明确引用，所以我们在任何时候调用，`self` 都指向实例对象 `tom`。而 JavaScript 中 `this` 并没有明确传入而是依赖于调用时的对象，但是我们可以明确地引用，这就造成了 `this` 的混乱。

你可能会说：我调用 `tom.bark()` / `bark()` 的时候并没有把实例对象(`tom`)作为参数传递进去呀？原因就在于[实例的方法是一个`属性描述符`](https://www.imliyan.com/blogs/article/Python%E4%B8%AD%E5%B1%9E%E6%80%A7%E5%A4%84%E7%90%86%EF%BC%88@property%E4%B8%8E%E6%8F%8F%E8%BF%B0%E7%AC%A6%EF%BC%89/#ps)而非定义时的那个函数对象。方法本身已经绑定了该实例，我们只用传递额外的参数，而方法会自动将实例对象传递给函数。

*在 JavaScript 中，我们可以使用 [class field](https://github.com/tc39/proposal-class-fields) 与箭头函数 `=>` 来实现实现 `this` 的自动绑定，但本文的目的不是为了实现 `this` 绑定，而是在此过程中来学习了解 Proxy、Reflect 等概念。*

## 什么是描述符

[Python 中的描述符](https://docs.python.org/3/howto/descriptor.html)就是一个实现了`__get__`/`__set__`/`__delete__`方法的对象。JavaScript 中的描述符则可以具有 `configurable	enumerable	value	writable	get	set` 等属性，分别构造出**数据描述符**/**存取描述符**。

我们来描述符来使 `bark` 方法成为一个绑定了实例参数的方法。

```js
class Dog {
    constructor(name) {
        this.name = name
    }
    bark(self) {
        console.log(`Wang! ${self.name}`)
    }
}

let tom = new Dog('Tom')

Object.defineProperty(tom, 'bark', {
    get() {
        return () => {
            return Dog.prototype.bark(tom)
        }
    }
})

tom.bark()
// Wang! Tom

let bark = tom.bark
bark()
// Wang! Tom
```

在我们读取 `tom.bark` 的时候，属性描述符的 getter 被执行，返回了一个新的函数，该函数不是我们在 Class 内定义的函数，而是一个包含了实例自由变量的闭包函数。当我们调用它的时候，它会自动把实例对象传递给 `Dog.prototype.bark` 方法。

用描述符（简单）实现了我们的目标，但是还有问题：

- 如果有多个方法，需要对每个方法定义一次描述符，很麻烦。
- 如果用户用 class field + 箭头函数的方法定义方法，name 方法将作为实例的属性，而非其原型的属性。`Dog.prototype.bark` 调用将会出错。如果我们在描述符内直接用 `tom.bark(tom)` 会导致重复无限递归查找：
    - `tom.bark --getter--> tom.bark --getter-->  tom.bark...`

## Proxy

现在我们来研究如何通过 Proxy 来实现。Proxy 就如其名字一样，可以代理(Proxy)对象的一些底层操作。

```js
var p = new Proxy(target, handler);
```

Proxy 的关键在 handler，handler 对象有多个陷阱(trap)用来处理代理对象的各种操作，包括属性读取、被删除、对象被调用、被 new、被判断属性的存在等等。如果没有对相应的行为定义 trap，那么对代理对象的操作将被转发到目标对象身上。

现在我们来拦截所有对方法的访问：如果对象属性是一个函数，我们就返回一个绑定了实例对象的高阶函数。

```js
class Dog {
    constructor(name) {
        this.name = name
    }
    bark(self) {
        console.log(`Wang! ${self.name}`)
    }
}

let methodHandler = {
    // target: 目标对象
    // property: 属性名
    // receiver: 代理对象
    get(target, property, receiver) {
        let value = Reflect.get(target, property)
        if (typeof value === 'function') {
            return (...args) => value(target, ...args)
        }
        return value
    }
}

let tom = new Dog('Tom')
let proxiedTom = new Proxy(tom, methodHandler)

proxiedTom.bark()
// Wang! Tom

let bark = proxiedTom.bark
bark()
// Wang! Tom

let snoopy = new Dog('Snoopy')
let proxiedSnoopy = new Proxy(snoopy, methodHandler)

proxiedSnoopy.bark()
// Wang! Snoopy
```

看起来我们不必为每个属性定义操作符了，而且我们调用 `proxiedTom.bark` 时不管 bark 方法在 tom 自己身上还是在其原型身上，都不会触发无限递归查找了：`proxiedTom.bark --proxy--> Reflect.get(tom, 'bark') -> bark method`

但是我们需要为每一个实例生成它的代理对象。可不可以在 `new Dog` 的时候就返回生成的代理对象呢？只需要代理 Dog 的 contruct 行为即可。

```js
class Cat {
    constructor(name) {
        this.name = name
    }
    mew(self) {
        console.log('Mew~', self.name)
    }
    eat(self, food) {
        console.log(self.name, 'eat', food)
    }
}

let contructHandler = {
    // target: 目标对象
    // argumentsList: 参数列表
    // newTarget: 最初被调用来构造的函数，如代理对象
    construct(target, argumentsList, newTarget) {
        // 构造实例对象
        let instance = Reflect.construct(target, argumentsList)
        // 代理并返回代理对象
        return new Proxy(instance, methodHandler)
    }
}

let ProxiedCat = new Proxy(Cat, contructHandler)

let tom = new ProxiedCat('Tom')
tom.mew()
// Mew~ Tom

let kitty = new ProxiedCat('Kitty')

kitty.mew()
// Mew~ Kitty

kitty.eat('milk')
// Kitty eat milk

kitty.name
// 'Kitty'

kitty.tomsmew()
kitty.tomsmew = tom.mew
// Mew~ Tom
```

## 那么 Reflect 又是什么？

Reflect 有一系列操作对象的静态方法。
每一个 handler trap 都对应一个 Reflect 操作，如 get trap 对应于 `Reflect.get`。defineProperty trap 对应于 `Reflect.defineProperty`。

有一些 Reflect 方法实现的效果与 Object 对象上的方法一样，但是有一些不同。

以 `Reflect.defineProperty` 为例，它的效果与 `Object.defineProperty` 一样，但是 `Reflect.defineProperty` 会在操作成功是返回布尔值 `true`，`Object.defineProperty` 则返回传递给它的对象。

```js
let tom = {
    name: 'tom'
}
let tommy = Object.defineProperty(tom, 'age', {
    get() {
        return 3
    }
})

tom === tommy
// true

let kitty = {
    name: 'kitty'
}
let result = Reflect.defineProperty(kitty, 'age', {
    get() { 
        return 3
    }
})

result === true
// true
```

关于 Reflect 提供的方法与 Object 的方法的具体不同，可以参考[这个表格](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/Comparing_Reflect_and_Object_methods)。

Reflect 鱼 Object 相比的另一个有用的地方是，在对象设置了 getter 的时候，Reflect 可以很好地处理这种情况。

```js
// 比如我们有一个会员对象
let member = {
    _name: 'Han Solo',
    get name() {
        return this._name
    }
}

// 然后我们想用一个代理来实现会员匿名。

let anonymousMember = new Proxy(member, {
    get(target, property, receiver) {
        if (property === '_name') {
            return 'anonymous'
        }
        return target[property]
    }
})

```

当我们调用 `anonymousMember.name`，会输出什么?

```
anonymousMember.name --proxy--> member['name'] ---getter--> this._name -> member._name -> 'Han Solo'
```

所以我们没有拦截到对 `member` 的 `_name` 属性的访问。

使用 Reflect 可以让我们指定 getter 内的 this 来实现对 `_name` 属性的真正拦截。

```js
let anonymousMember = new Proxy(member, {
    get(target, property, receiver) {
        if (property === '_name') {
            return 'anonymous'
        }
        // 第三个参数 receiver 指定了 getter 调用时的 this，在此为我们的代理对象 anonymousMember
        return Reflect.get(target, property, receiver)
    }
})

anonymousMember.name
// 'anonymous'

// anonymousMember.name --proxy--> Reflect.get(member, 'name', anonymousMember) ---getter---> anonymousMember._name -> 'anonymous'
```

---

## 参考

- [Python Descriptor HowTo Guide](https://docs.python.org/3/howto/descriptor.html)
- [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [Reflect](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect)
- [understanding es6: Proxies and the Reflection API](https://github.com/nzakas/understandinges6/blob/master/manuscript/12-Proxies-and-Reflection.md)

