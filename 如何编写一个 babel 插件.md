前一段时间项目上有过一个需求，可以考虑通过自动给 React 组件添加属性来实现。虽然这是一个很反模式的方式并且最终否决了这个方案，但是还是尝试研究了一下如何实现 `自动给 React 组件添加属性`。更确切地说，是**在 webpack 打包出来的最终文件内给 React 组件添加上自定义的 props**。

webpack 处理 React 文件（js/jsx）使用 babel-loader，babel 就是我们的 JavaScript 编译器，它接收我们的源代码作为输入，产出编译后的可运行于浏览器的目标代码作为输出。babel 支持插件（plugin），可以视作编译器前端与后端之间的中间件：前端根据源代码生成抽象语法树（AST）等，后端根据抽象语法树生成目标代码，而插件作为中间件则是在生成目标代码之前对抽象语法树做相应的修改。

- [基础概念](#%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5)
  - [抽象语法树](#%E6%8A%BD%E8%B1%A1%E8%AF%AD%E6%B3%95%E6%A0%91)
  - [访问者模式](#%E8%AE%BF%E9%97%AE%E8%80%85%E6%A8%A1%E5%BC%8F)
- [插件编写](#%E6%8F%92%E4%BB%B6%E7%BC%96%E5%86%99)
- [我应该使用这个插件吗？](#%E6%88%91%E5%BA%94%E8%AF%A5%E4%BD%BF%E7%94%A8%E8%BF%99%E4%B8%AA%E6%8F%92%E4%BB%B6%E5%90%97)

## 基础概念

### 抽象语法树

`@babel/parser` 模块用来接收我们的源代码来生成抽象语法树，

```ts
import * as parser from '@babel/parser'

let code = 'let result = x + y'

let ast = parser.parse(code)

console.log(ast)
```

将会输出

```
Node {
  type: 'File',
  start: 0,
  end: 13,
  loc:
   SourceLocation {
     start: Position { line: 1, column: 0 },
     end: Position { line: 1, column: 13 } },
  program:
   Node {
     type: 'Program',
     start: 0,
     end: 13,
     loc: SourceLocation { start: [Position], end: [Position] },
     sourceType: 'script',
     interpreter: null,
     body: [ [Node] ],
     directives: [] },
  comments: [] }
```

如果我们打印出 `ast.program.body`，我们会得到：

```
[ Node {
    type: 'VariableDeclaration',
    start: 0,
    end: 18,
    loc: SourceLocation { start: [Position], end: [Position] },
    declarations: [ [Node] ],
    kind: 'let' } ]
```

可以看到抽象语法树是由一个个 Node 组成的，每个 Node 有很多属性，而 Node 还会有一些 body、left 之类的属性，这些属性的值的类型也可能是 Node。对抽象语法树的修改就是修改这些 Node 的值或者属性。

### 访问者模式

> 访问者模式是一种将算法与对象结构分离的软件设计模式。
>
> 这个模式的基本想法如下：首先我们拥有一个由许多对象构成的对象结构，这些对象的类都拥有一个accept方法用来接受访问者对象；访问者是一个接口，它拥有一个visit方法，这个方法对访问到的对象结构中不同类型的元素作出不同的反应；在对象结构的一次访问过程中，我们遍历整个对象结构，对每一个元素都实施accept方法，在每一个元素的accept方法中回调访问者的visit方法，从而使访问者得以处理对象结构的每一个元素。我们可以针对对象结构设计不同的实在的访问者类来完成不同的操作。
> 
> ———— 维基百科

具体来说，我们的 AST 的每一个 Node 有一个 accept 方法，当我们用一个 visitor 来遍历我们的 AST 时，每遍历到一个 Node 就会调用这个 Node 的 accept 方法来 `接待` 这个 visitor，而在 accept 方法内，我们会回调 visitor 的 visit 方法。我们来用访问者模式来实现一个 `旅行者访问城市景点` 的逻辑。

**实际上 Node 是有两个方法，enter 和 exit，指遍历进入和离开 Node 的时候。通常访问者的 visit 方法会在 enter 内被调用。*

```js
// 旅游景点
class ScenicPoint {
    constructor(name) {
        this.name = name
    }
    // 景点的 accept 方法接收 visitor，函数内调用 visitor 的 visit 方法来 visit 景点的实例
    accept(visitor) {
        visitor.visit(this)
    }
}

class Park extends ScenicPoint { }

class Museum extends ScenicPoint { }

// 我们的城市
class City {
    constructor(name, scenicPointList) {
        this.name = name
        this.scenicPointList = scenicPointList
    }
    accept(visitor) {
        for (let scenicPoint of this.scenicPointList) {
            scenicPoint.accept(visitor)
        }
    }
}

// visitors: Alice 与 Bob
let Alice = {
    name: 'Alice',
    visit(scenicPoint) {
        if (scenicPoint instanceof Park) {
            console.log(`${scenicPoint.name} is a wonderful park~`)
        } else {
            console.log(`${this.name} visiting ${scenicPoint.name}`)
        }
    }
}
let Bob = {
    name: 'Bob',
    visit(scenicPoint) {
        if (!(scenicPoint instanceof Museum)) {
            console.log(`I want to go to some Museum.`)
        } else {
            console.log(`${scenicPoint.name} is a wonderful Museum~`)
        }
    }
}

let BeiJing = new City('BeiJing', [
    new ScenicPoint('国家博物馆'),
    new Park('玉渊潭公园'),
    new Museum('故宫'),
])
```

我们用一个访问者来访问北京：

```js
BeiJing.accept(Alice)
// Alice visiting 国家博物馆
// 玉渊潭公园 is a wonderful park~
// Alice visiting 故宫

BeiJing.accept(Bob)
// I want to go to some Museum.
// I want to go to some Museum.
// 故宫 is a wonderful Museum~
```

可以发现，我们可以自定义 visitor 的 visit 方法，针对不同的景点实现不同的逻辑。

返回到我们的 babel 插件，每一个 Node 都可以视为一个景点，而我们的插件则是一个 visitor，我们在插件内定义了针对不同 Node 类型的处理方式，当遍历到对应的节点时调用对应的方法进行处理。

## 插件编写

首先我们应该看我们的 React 元素是怎么创建出来的：

```js
let element = <Button color="red" />
```

会被编译为：

```js
var element = _react.default.createElement(Button, {
  color: "red"
})
```

可以看到组件属性作为 `_react.default.createElement` 的第二个参数，我们只需要修改这个参数值就可以了。

我们可以先看上述代码生成的 AST：

```js
{ type: 'CallExpression',
  callee:
   { type: 'MemberExpression',
     object: { type: 'Identifier', name: 'React' },
     property: { type: 'Identifier', name: 'createElement' },
     computed: false,
     optional: null },
  arguments:
   [ Node {
       type: 'Identifier',
       start: 42,
       end: 48,
       loc: [SourceLocation],
       name: 'Button' },
     { type: 'ObjectExpression', properties: [Array] } ],
  typeAnnotation: undefined,
  typeParameters: undefined,
  returnType: undefined,
  start: 41,
  loc:
   SourceLocation {
     start: Position { line: 3, column: 14 },
     end: Position { line: 3, column: 36 } },
  end: 63,
  trailingComments: [],
  leadingComments: [],
  innerComments: [] }
```

可以发现 props 是这个 CallExpression 的 arguments 属性数组的第二个元素，而 props 自身是一个 ObjectExpression，我们只需要用我们的 props 对象来替换/合并原来的 ObjectExpression 即可。

思路已有，直接上最终代码：

```ts
// parser
import * as parser from '@babel/parser'
// 各种 Node 类型定义
import * as types from '@babel/types'
import traverse, { NodePath } from '@babel/traverse'

export default function() {
    return {
        // 我们的 visitor
        visitor: {
            // 针对函数调用的单独逻辑处理
            CallExpression(path: NodePath<types.CallExpression>, state) {
                // 我们只处理 React.createElement 函数调用
                let { callee } = path.node
                if (
                    !(
                        types.isMemberExpression(callee) &&
                        types.isIdentifier(callee.property) &&
                        callee.property.name === 'createElement'
                    )
                ) {
                    return
                }

                // 从第一个参数获取组件名称（Button）
                // 从第二个参数获取组件属性
                let [element, propsExpression] = path.node.arguments
                let elementType: string
                if (types.isStringLiteral(element)) {
                    elementType = element.value
                } else if (types.isIdentifier(element)) {
                    elementType = element.name
                }

                // 我们的插件支持自定义选项，针对不同的组件类型传入不同的额外自定义属性
                const options: Object = state.opts
                let extraProps: Object | undefined = options[elementType]

                // 如果没有针对次组件类型的额外参数，我们的插件什么都不做
                if (!extraProps) {
                    return
                }

                // 否则，我们利用 parser.parseExpression 方法以及我们的自定义属性生成一个 ObjectExpression
                let stringLiteral = JSON.stringify(extraProps)
                let extraPropsExpression = parser.parseExpression(stringLiteral)

                // 如果组件原本 props 为空，我们直接将我们的自定义属性作为属性参数
                if (types.isNullLiteral(propsExpression)) {
                    path.node.arguments[1] = extraPropsExpression
                } else {
                // 否则，我们将我们的自定义属性与原属性进行合并
                    path.node.arguments[1] = types.objectExpression(
                        (<types.ObjectExpression>propsExpression).properties.concat(
                            (<types.ObjectExpression>extraPropsExpression).properties,
                        ),
                    )
                }
            },
        },
    }
}
```

然后我们修改 babel 的配置文件启用我们的插件：

```js
// babel.config.js
const plugins = [
    [
        'babel-plugin-react-auto-props',
        {
            'Button': {
                size: 'small',
            },
        },
    ],
]

module.exports = { presets, plugins }
```

然后我们的源代码为：

```jsx
let button = <Button type="primary" />
```

使用 babel 编译后的结果为：

```js
var button = _react.default.createElement(Button, {
  type: "primary",
  "size": "small"
});
```

可以看到属性已经自动添加成功。

## 我应该使用这个插件吗？

这个插件已经发布到 npm，不过请注意，该插件只是一个实验性的插件，而且它的结果很反模式。如果在项目中确实需要为组件自动添加属性，请使用[特例关系组件](https://zh-hans.reactjs.org/docs/composition-vs-inheritance.html#specialization)。

```jsx
class SmallButton extends Component {
    render() {
        return <Button {...this.props} size="small" />
    }
}
```

如果只是想尝试，可以通过 npm 安装：

```
npm install -D babel-plugin-react-auto-props
```