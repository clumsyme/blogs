在不同的语境中，`this`代表不同的对象。

## 全局上下文

在全局上下文中（任何函数体外）， `this` 指代全局对象。

    :::javascript
    console.log(this === window);
    // true

## 函数上下文

在函数内部， `this`的值取决于函数是如何被调用的。

### 直接调用

    :::javascript
    function say() {
        return this
    }
    say() === window
    // true

而在严格模式下，this在进入运行环境时设置，将是undefined。

    :::javascript
    function say2() {
        "use strict";
        return this
    }
    say2() === undefined
    // true

### 作为方法

如果函数作为一个对象的方法，则`this`指代调用此方法的对象。注意箭头函数在{}定义对象
中的不同表现。

    :::javascript
    var dog = {
        name: "Tom",
        bark: function() {
            return this
        },
        walk: () => {
            return this
        },
        head: {
            name: "Tom's head",
            shake: function() {
                return this.name + " shaking"
            }
        }
    }
    dog.bark() === dog
    // true
    dog.walk() === window
    // true

即使方法是之后才指定为对象的属性。

    :::javascript
    dog.run = function() {
        return this.name + " running"
    }
    dog.run()
    // Tom running

`this`指向其最近的对象。

    :::javascript
    dog.head.shake()
    // Tom's head shaking

 另外构造函数中的函数(包括箭头函数)则指向产生的对象本身。

    :::javascript
    function Dog() {
        this.walk = () => {
            return this
        }
        this.bark = function() {
            return this
        }
    }
    var d = new Dog()
    d.bark() === d
    // true

    d.walk === d
    // true

### call 与 apply、bind

call() 方法可用于指定函数体内的`this`对象。
apply() 指定`this`，并且第二个数组参数的各项作为函数的参数。

    :::javascript
    function greet(greeting) {
        return greeting + this.fname + " " + this.lname
    }
    var tom = {fname: "Tom", lname: "Jerry"}
    greet.call(tom, "Hello ")
    // Hello Tom Jerry
    greet.apply(tom, ["Hi~ "])
    // Hi~ Tom Jerry

bind()方法返回一个新的函数，函数体与函数相同，但`this`被永久地绑定。

    :::javascript
    greet = greet.bind(tom)
    greet("Hello ")
    // Hello Tom Jerry

### 箭头函数

对于箭头函数，`this`值一旦确定，就会一直保持不变。

    :::javascript
    var name = "Jerry",
    say = (() => this.name),
    tom = {
        name: "Tom",
        say: say,
        walk: function() {
            var inwalk = () => this.name + " walking"
            return inwalk
        }
    }

    say() === "Jerry"
    // true

    tom.say() === "Jerry"
    // true

    say.call(tom) === "Jerry"
    // true

    say.bind(tom)() === "Jerry"
    // true

    var globalwalk = tom.walk()
    globalwalk()
    // Tom walking

    globalwalk.bind(window)
    globalwalk()
    // Tom walking

### DOM事件处理函数中的`this`

当函数作为DOM时间处理函数时，`this`指向触发事件的元素。

    :::javascript
    var element = document.getElementById('id1')
    element.onclick = function(e) {
        console.log(this === e.currentTarget) // 总是 true
        console.log(this === e.target) // 当currenrTarget === target
    }

### 行内事件处理函数中的`this`

```html
<button onclick="alert(this)">Click</buttom>
```

上述代码的`this`指向`<button>` DOM对象。
注意只有外层的`this`被如此设置。

```html
<button onclick="alert((function(){return this})())"
```

上述代码的`this`未指定所以其值默认为全局对象(window / global)。