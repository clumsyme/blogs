最近学习 SwiftUI，看到有如下写法：

```swift
struct ContentView: View {
    var body: some View {
        VStack(alignment: .leading) {
            Text("Trutle Rock")
                .font(.title)
                .foregroundColor(.black)
            HStack {
                Text("Joshua Tree National Park")
                    .font(.subheadline)
                Spacer()
                Text("California")
                    .font(.subheadline)
            }
        }
    }
}
```

其中

```swift
VStack(alignment: .leading) {
    //...
}
```

这个闭包语句，一开始没有很好理解。它的语法看起来像函数声明语句，另外它和[计算属性](https://docs.swift.org/swift-book/ReferenceManual/Declarations.html#ID358)有什么区别？为什么可以写到实例化语句的后边？

## 闭包

定义上，闭包是实现[词法作用域](<https://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scoping>)的一种手段。在 Python、JavaScript 中都有闭包。简单来说，闭包就是一个保留了其定义时环境变量的函数，即使该环境已经不复存在。变量作为自由变量存在于函数内，就像被函数**捕获**了一样（参见[Python 中的闭包与局部作用域](https://www.imliyan.com/blogs/article/Python%E4%B8%AD%E7%9A%84%E9%97%AD%E5%8C%85%E4%B8%8E%E5%B1%80%E9%83%A8%E4%BD%9C%E7%94%A8%E5%9F%9F/)）。

通常会把一个在局部定义的函数成为闭包。

```bash
|—————————————————|
|   scope A       |
|   x             |
|                 |
|     |————————|  |        |————————|
|     | scopeB |  |        | scopeB |
|     | x      |--|------> |        |
|     |        |  |        | |      |
|     |————————|  |        |—|——————|
|                 |          |
|—————————————————|          x (as free variable)
```

## Swift 中的闭包

Swift 中的闭包也不例外，它是*自包含的函数代码块，可以在代码中被传递和使用，闭包可以捕获和存储其所在上下文中任意常量和变量的引用*。

Swift 中有三种形式的闭包：

1. 全局函数：一个有名字但不捕获任何值的闭包。
2. 嵌套函数：一个有名字且可以捕获其封闭函数域内值的闭包。
3. 闭包表达式：一个轻量级的可以捕获其上下文变量及常量的匿名闭包。

全局函数和嵌套函数闭包在 JavaScript 等语言内也是很常见的，在此我们就不再做进一步的研究，这里我们主要研究的是第三中形式：闭包表达式。

### 闭包表达式

[闭包表达式](https://docs.swift.org/swift-book/ReferenceManual/Expressions.html#ID393)的语法结构如下：

```
{ (parameters) -> (return type) in
    statements
}
```

一个例子：

```swift
let names = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]

names.sorted(by: { (s1: String, s2: String) -> Bool in
    return s1 > s2
})
```

我们定义了一个闭包作为 `by` 参数的值。

闭包表达式可以有以下几个更简洁的形式：

#### 闭包表达式可以省略其参数或者返回值的类型

```swift
names.sorted(by: { s1, s2 in return s1 > s2 })
```

#### 闭包表达式可以省略其参数名字，然后用`$0、$1`依次来引用参数

如果参数名字和类型都省略了，`in` 关键字也可以省略。

```swift
names.sorted(by: {return $0 > $1})
```

#### 如果返回值只是一个简单的语句，那么 `return` 也可以省略

```swift
names.sorted(by: {$0 > $1})
```

### 尾随闭包

如果需要将一个闭包表达式作为一个函数的最后一个参数，而这个闭包表达式又很长，可以使用尾随闭包来增加可读性。尾随闭包写在函数括号后，即使它是函数的一个参数。

下边我们看 SwiftUI 的 `VStack` 定义：

```swift
public struct VStack<Content> : View where Content : View {
    @inlinable public init(alignment: HorizontalAlignment = .center, spacing: CGFloat? = nil, @ViewBuilder content: () -> Content)

    public typealias Body = Never
}
```

可以看到其构造器最后一个参数接收一个返回 `Content` 类型的闭包，所以文章开头提到的代码其实就是尾随闭包。

```swift
VStack(alignment: .leading) {
    Text("Trutle Rock")
        .font(.title)
        .foregroundColor(.black)
    HStack {
        Text("Joshua Tree National Park")
            .font(.subheadline)
        Spacer()
        Text("California")
            .font(.subheadline)
    }
}
```

如果尾随闭包是函数的唯一参数，则函数括号也可以省略：

```swift
names.sorted() {$0 > $1}

// equals

names.sorted {$0 > $1}
```

### 捕获值

捕获值是闭包最基础的性质。利用这一性质，我们可以方便的写出一些高阶函数。

```swift
func makeIncrementer(_ amount: Int) -> () -> Int {
    var total = 0
    func incrementer() -> Int {
        total += amount
        return total
    }
    return incrementer
} 

let incrementOne = makeIncrementer(1)
let incrementTwo = makeIncrementer(2)

print(incrementOne()) // 1
print(incrementOne()) // 2
print(incrementOne()) // 3

print(incrementTwo()) // 2
print(incrementTwo()) // 4
print(incrementTwo()) // 6
```

### 闭包是引用类型

注意上述 `increOne` 和 `increTwo` 都是常量，但是闭包仍然可以修改其捕获的 `total` 变量。这是因为函数和闭包是引用类型。当我们把闭包赋值给一个常量或者变量时，指向闭包的 `increOne` 是常量或者变量，而非闭包的内容。

```swift
let anotherIncrement = incrementOne
print(anotherIncrement()) // 4
```

### 逃逸闭包

如果一个闭包作为参数传递给函数，但是在函数返回之后才执行，就称这个闭包从函数中*逃逸*。我们可以在参数名前添加 `@escaping` 来标注允许这个闭包逃逸。

```swift
var completionHandlers: [() -> Void] = []
func someFunctionWithEscapingClosure(completionHandler: @escaping () -> Void) {
    completionHandlers.append(completionHandler)
}
```

逃逸闭包必须明确引用 `self`:

```swift
func someFunctionWithNonescapingClosure(closure: () -> Void) {
    closure()
}

class SomeClass {
    var x = 10
    func doSomething() {
        // 必须明确引用 self
        someFunctionWithEscapingClosure { self.x = 100 }
        // 不必明确引用 self
        someFunctionWithNonescapingClosure { x = 200 }
    }
}

let instance = SomeClass()
instance.doSomething()
print(instance.x)
// Prints "200"

completionHandlers.first?()
print(instance.x)
// Prints "100"
```

### 自动闭包

如果一个闭包不接收任何参数，调用时返回闭包内的表达式，我们可以添加 `@autoclosure` 来将表达式自身作为闭包。

```swift
func autoClosure(closure: @autoclosure () -> Int ) -> Int {
    reeturn closure()
}

print(autoClosure(closure: 9)) // 9

// 对比非自动闭包
func nonAutoClosure(closure: () -> Int ) -> Int {
    reeturn closure()
}

print(nonAutoClosure(closure: { 9 })) // 9
```

自动闭包可以省略花括号，但是过度使用会降低代码的可读性。

参考：

- [Closure - Swift Language Guide](https://docs.swift.org/swift-book/LanguageGuide/Closures.html)
- [Expressions - Swift Language Reference](https://docs.swift.org/swift-book/ReferenceManual/Expressions.html)
- [Statements - Swift Language Reference](https://docs.swift.org/swift-book/ReferenceManual/Statements.html)
- [Declarations - Swift Language Reference](https://docs.swift.org/swift-book/ReferenceManual/Declarations.html)