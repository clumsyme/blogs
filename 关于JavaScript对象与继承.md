## 简述JavaScript对象与继承

JavaScript是基于原型的语言，每一个对象都有一个原型对象。通过构造器函数实例化的对象可以通过浏览器实现的`__proto__`接口访问其原型对象(ECMA官方实现为[`Object.getPrototypeOf(obj)`](http://www.ecma-international.org/ecma-262/6.0/#sec-object.getprototypeof))，此对象的构造器函数同样有一个属性
`prototype`指向此原型对象。

当查找对象属性时，会首先在对象上查找，如果找到则返回，未找到则查找对象的原型对象、原型对象的原型对象……一直到查找到为止或者某对象的原型对象为`null`。

如果要实现继承，可以指定子对象的原型是父对象的一个实例，也即：

    :::JavaScript
    SubType.prototype = new SuperType()

对象与继承的基础知识本文不再详述，接下来对比构造函数与class两种语法格式创建对象的特点。

## 使用构造器函数

    :::JavaScript
    function Dog(name) {
        this.name = name
        this.bark = function() {
            console.log(this.name + ' Wang!')
        }
    }
    Dog.prototype.walk = function() {
        console.log(this.name + ' Walking!')
    }

    var d = new Dog('Tom')

    // name, bark 是对象 d 的属性
    d.hasOwnProperty('name')
    // true
    d.hasOwnProperty('bark')
    // true

    // walk 不是对象 d 的属性
    d.hasOwnProperty('walk')
    // false

    d.__proto__.hasOwnProperty('walk')
    // true
    d.__proto__ === Dog.prototype
    // true
    d.walk()
    // 'Tom Walking!'

### 实现继承

    :::JavaScript
    function Doggy(name, age) {
        Dog.call(this, name)
        this.age = age
    }
    var dg = new Doggy('Doggy', 3)

    dg.hasOwnProperty('name')
    // true
    dg.hasOwnProperty('bark')
    // true

    // 我们虽然在 Doggy 的上下文调用了 Dog 构造函数，
    // 但这只是一个新的构造函数， 我们还未实现继承
    // 因为 Doggy 的 prototype 未定义，Doggy 未继承 Dog
    dg.walk()
    // TypeError: dg.walk is not a function

    // 指定原型实现继承
    Doggy.prototype = Object.create(Dog.prototype)
    var dg = new Doggy('Doggy', 3)
    dg.walk()
    // 'Doggy Walking!'

    // dg原型的原型是 d 的原型
    // dg 的原型是 Dog 的实例
    dg.__proto__.__proto__ === d.__proto__
    // true
    dg.__proto__ instanceof Dog
    // true

## ECMAScript 6 之 class

class 在ECMAScript 6中引进，但这只是一个语法糖，JavaScript仍是一个基于原型而不是基于类的语言。

    :::JavaScript
    class Dog {
        constructor(name) {
            this.name = name
        }
        bark() {
            console.log(this.name + ' Wang!')
        }
    }

    class Doggy extends Dog {}

    // Dog 其实是一个 function 实例
    typeof Dog
    // 'function'

    var d = new Dog('Tom')
    var dg = new Doggy('Doggy')

    d.hasOwnProperty('name')
    // true

    // d 自身没有 bark 属性，是在其原型上查找到的
    d.hasOwnProperty('bark')
    // false
    d.__proto__.hasOwnProperty('bark')
    // true
    d.__proto__ === Dog.prototype
    // true

    dg.hasOwnProperty('name')
    // true
    dg.hasOwnProperty('bark')
    // false
    dg.__proto__.hasOwnProperty('bark')
    // false

    // ***********************
    //
    // dg 对象的原型是 Dog 的实例
    // dg 原型的原型就是d的原型
    //
    // ***********************
    dg.__proto__ instanceof Dog
    // true
    dg.__proto__.__proto__ === d.__proto__
    // true

两种构造方法对比可见，其内部实现是一致的， 但是`class`声明更加清晰明确，特别是对于熟悉其他基于类的语言的同学来说更容易理解。但必须要明白的是，这只是一个语法糖，JavaScript 仍是一个基于原型的语言。

参考

---

+ JavaScrip高级程序设计(第三版)

+ [Object prototypes - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects/Object_prototypes)

+ [Inheritance in JavaScript - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects/Inheritance)