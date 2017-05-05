## 问题：`[]`等于`false`么

看下边的表达式：

    :::javascript
    [] == false
    //  true

    if ([]) {
        console.log('啦啦啦')
    }
    // 啦啦啦


额？`[]`不是等于`false`么？
而且，再看这里：

    :::js
    ![] == false
    // true

=_=!

## 究竟相等不相等？

不过这不是bug，而是由于JavaScript语言对`==`的处理方式引起的。

### JavaScript相等性判断

1. `==`

2. `===`

3. `Object.is`

先说后两个：

### 全等操作符`===`

>全等操作符比较两个值是否相等，两个被比较的值在比较前都不进行隐式转换。如果两个被比较的值具有不同的类型，这两个值是不全等的。否则，如果两个被比较的值类型相同，值也相同，并且都不是 number 类型时，两个值全等。最后，如果两个值都是 number 类型，当两个都不是 NaN，并且数值相同，或是两个值分别为 +0 和 -0 时，两个值被认为是全等的。
>
>在日常中使用全等操作符几乎总是正确的选择。对于除了数值之外的值，全等操作符使用明确的语义进行比较：一个值只与自身全等。对于数值，全等操作符使用略加修改的语义来处理两个特殊情况：第一个情况是，浮点数 0 是不分正负的。区分 +0 和 -0 在解决一些特定的数学问题时是必要的，但是大部分境况下我们并不用关心。全等操作符认为这两个值是全等的。第二个情况是，浮点数包含了 NaN 值，用来表示某些定义不明确的数学问题的解，例如：正无穷加负无穷。全等操作符认为 NaN 与其他任何值都不全等，包括它自己。（等式 (x !== x) 成立的唯一情况是 x 的值为 NaN）

### `Object.is()`

>Object.is 在三等号判等的基础上特别处理了 NaN 、 -0 和 +0 ，保证 -0 和 +0 不再相同，但 Object.is(NaN, NaN) 会返回 true。

### 非严格相等`==`

与全等操作符不同的是，非严格相等操作符在比较两个值`A == B`的时候，如果两者类型不同，会先进行类型转换，转换为同一类型后再进行比较。具体的转换规则如下：


| |Undefined|Null|Number|String|Boolean|Object|
|---|---|---|---|---|---|---|
Undefined|true|true|false|false|false|IsFalsy(B)
Null|true|true|false|false|false|IsFalsy(B)
Number|false|false|A === B|A === ToNumber(B)|A=== ToNumber(B)|A=== ToPrimitive(B) 
String|false|false|ToNumber(A) === B|A === B|ToNumber(A) === ToNumber(B)|ToPrimitive(B) == A
Boolean|false|false|ToNumber(A) === B|ToNumber(A) === ToNumber(B)|A === B|false
Object|false|false|ToPrimitive(A) == B|ToPrimitive(A) == B|ToPrimitive(A) == ToNumber(B)|A === B

>*ToNumber(A) 尝试在比较前将参数 A 转换为数字，这与 +A（单目运算符+）的效果相同。通过尝试依次调用 A 的A.toString 和 A.valueOf 方法，将参数 A 转换为原始值（Primitive）。*

第一列为被比较值A的类型，第一排为被比较值B的类型。

#### 关于`[] == false`
如此我们看`[] == false`，`[]`是`Object`类型，`false`是`Boolean`类型，所以真正的比较表达式是：

    ToPrimitive([]) == ToNumber(false)

`ToPrimitive([])`通过调用[].toString()得到空字符串`""`，`ToNumber(false)`将`false`转换为数字`0`，因此：


    "" == 0


然后是字符串与数字进行非严格相等比较，又进行了类型转换：

    ToNumber("") === 0

也就是：

    0 === 0

因此返回`true`。

所以，**`[] == false`表达式的值为`true`，但空数组最想本身`[]`是不被视为`false`的。实际上，JavaScript中视为`false`的有`false, null, undefined, 0, "", NaN`，其他类型可通过类型转换转换为`Boolean`类型**。

#### 关于`![] == false`

逻辑非操作符`!`的优先级(15)高于等号操作符优先级(10)，因此在比较前先进行的是`![]`，结果是`false`，因此实际比较是：

    false == false

因此返回`true`。

## 不是bug的bug

> Explicit is better than implicit.
>
> Readability counts.

好的代码应该是清晰明了、可读性好的，语言最好也是。而在一个相等操作符`==`上，JavaScript进行了如此多的内部操作，不仅新手容易迷惑，就连有经验的程序员也可能一时搞不明白一个表达式到底出了什么问题。因此也有了如下观点：

>最好永远都不要使用相等操作符。全等操作符的结果更容易预测，并且因为没有隐式转换，全等比较的操作会更快。

参考：
---
- [Equality comparisons and sameness](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness)

- [Operator precedence](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence)

- [Falsy](https://developer.mozilla.org/en-US/docs/Glossary/Falsy)
