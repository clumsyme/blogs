## 为什么使用 CSS Variables

在开发一个 Web 应用的时候，经常需要为许多元素设置属性（如`color`、`width`、`height`）等，即使很多元素具有相同的属性，我们还是要为每个元素单独设置属性。这样一旦应用更改主题，比如更改主色调，那么要为每一个元素一个一个地修改，为什么不能像大部分编程语言一样，定义一个变量，用这个变量得引用来设置对象（元素）的属性？这样只需要修改变量值就可以修改所有的属性值了。CSS Variables 就是做这个的。

## 为什么不用SASS/LESS

像 SASS 和 LESS 这些 CSS 预处理语言也支持变量定义等多种有用特效，它们是很有用的工具，但是这些工具有一个明显的缺点：它们是静态的，最终还是要编译为CSS，不能在运行时进行更改，也就不能动态的运用主题更改或者进行某些响应式设计。而 CSS Variables 是 CSS 的原生属性，没有这些问题，同时结合*未来可能出现的*新的 CSS 特性，它可以发挥出更大的作用。

## 浏览器支持情况

具体的支持情况可在[caniuse](http://caniuse.com/#search=css%20v)查看，目前除IE外，所有主流浏览器都实现了对 CSS Variables 的支持。

## CSS Variables 基本用法

### 定义变量

变量的定义格式是`--somevar`，如下`--text-color: red`，即将`red`的值赋给`--text-color`。

要使用定义的变量，使用`var()`函数，如下`var(--text-color)`会返回`--text-color`对应的值，也即是`red`。

CSS:

    :::css
    :root {
        --text-color: red
    }
    .cv-test {
        color: var(--text-color)
    }

HTML：

    :::html
    <p> class='cv-test'>May The force be with you.</p>

结果：

<style>
:root {
    --text-color: red;
    --backup-color: skyblue
}
.cv-test {
    color: var(--text-color)
}
</style>

---

<p class='cv-test'>May The force be with you.</p>

---

## 支持作用域与继承

CSS Variables 的另一个特性是支持作用域，这就意味着我们可以在不同的层级定义相同名称的变量。如果某元素没有 CSS Variables，则会继承父元素。

利用这一特性，我们也可以通过媒体查询写出响应式的样式来。

CSS:

    :::css
    .scopeone {
        --text-color: blue
    }

    .scopetwo {
        --text-color: orange
    }

    .cv-test {
        color: var(--text-color)
    }

    /*结合媒体查询定义变量*/
    @media (max-width: 1000px) {
        :root {
            --text-color: lime
    }
}

HTML:

    :::html
    <p class='cv-test'>My name is Ozymandias</p>

    <div class='scopeone'>
        <p class='cv-test'>
            <span>King<span> of kings
        </p>
    </div>

    <div class='scopetwo'>
        <p class='cv-test'>Look on my work</p>
    </div>

<style>
.scopeone {
    --text-color: green
}

.scopetwo {
    --text-color: orange
}

.cv-test {
    color: var(--text-color)
}

@media (max-width: 1000px) {
    :root {
        --text-color: lime
    }
}
</style>

结果：

---

<p class='cv-test'>My name is Ozymandias</p>

<div class='scopeone'>
    <p class='cv-test'>
        <span>King<span> of kings
    </p>
</div>

<div class='scopetwo'>
    <p class='cv-test'>Look on my work</p>
</div>

---

## 关于`var()`函数

`var()`函数的语法如下：

    :::css
    var(<custom-property-name> [, <declaration-value> ]? )

`custom-property-name`也就是我们预先定义的变量名，`declaration-value`则是在`custom-property-name`不存在或不能正常解析是作为备选值。

CSS:

    :::css
    .scopethree {
        color: var(--undefined-color, yellow) 
    }

HTML:

    :::html
    <p class='scopethree'>One ring to tule them all</p>

<style>
.scopethree {
    color: var(--undefined-color, yellow) 
}
</style>

结果：

---

<p class='scopethree'>One ring to rule them all</p>

---

### `var()`函数几个注意点

#### 不能将`var()`函数返回的值作为变量名：

    :::css
    /*错误用法*/
    :root {
        --mt: margin-top;
        var(--mt): 10px; 
    }

#### 不能将`var()`函数返回的值作为部分变量值：

    :::css
    /*错误用法*/
    :root {
        --big: 20;
        font-size: var(--big)px; 
    }

如果属性值是由变量计算的来，最好结合`calc()`函数使用：

CSS:

    :::css
    .scopefour {
        --big: 20;
        font-size: calc((var(--big) + 5) * 1px); 
    }

HTML:

    :::html
    <p class='scopefour'>One ring to find them</p>

<style>
.scopefour {
    --big: 20;
    font-size: calc((var(--big) + 5) * 1px); 
}
</style>

结果：

---

<p class='scopefour'>One ring to find them</p>

---

## 使用 JS 动态设定 CSS 样式

前文提到过，CSS Variables 相比 SASS/LESS 的一个优点是可以动态修改，我们可以通过 JS 来动态改变 CSS Variables:

打开控制台，输入以下代码：

    :::js
    getComputedStyle(document.documentElement).getPropertyValue('--text-color')

就可以得到我们在`:root`作用域内设置的`--text-color`变量值，这里就是`red`。

然后可以通过下列代码设置变量值，就可以做到批量修改属性值了：

    :::js
    document.documentElement.style.setProperty('--text-color', 'pink');

    // 或者使用另一个变量值来作为setProperty的参数
    document.documentElement.style.setProperty('--text-color', 'var(--backup-color)');

实例：

    :::html
    <p class='cv-test'>one ring to rule them all</p>
    <p class='cv-test2'>one ring to find them</p>
    <p class='cv-test3'>one ring to rule them all</p>
    <p class='cv-test4'>and in the darkness bind them</p>
    <button id='topurple'>紫色</button>
    <button id='toskyblue'>天蓝色</button>
    <style>
        cv-test2 {
            color: var(--text-color)
        }
        cv-test3 {
            color: var(--text-color)
        }
        cv-test4 {
            color: var(--text-color)
        }
    </style>
    <script>
        document.getElementById('topurple').addEventListener('click', () => {
            document.documentElement.style.setProperty('--text-color', 'purple');
        })
        document.getElementById('toskyblue').addEventListener('click', () => {
            document.documentElement.style.setProperty('--text-color', 'var(--backup-color)');
        })
    </script>


结果：

---

<p class='cv-test'>one ring to rule them all</p>
<p class='cv-test2'>one ring to find them</p>
<p class='cv-test3'>one ring to rule them all</p>
<p class='cv-test4'>and in the darkness bind them</p>
<button id='topurple'>紫色</button>
<button id='toskyblue'>天蓝色</button>

<style>
.cv-test2 {
    color: var(--text-color);
}

.cv-test3 {
    color: var(--text-color);
}

.cv-test4 {
    color: var(--text-color);
}
</style>

<script>
    document.getElementById('topurple').addEventListener('click', function() {
        document.documentElement.style.setProperty('--text-color', 'purple');
    })
    document.getElementById('toskyblue').addEventListener('click', function() {
        document.documentElement.style.setProperty('--text-color', 'var(--backup-color)');
    })
</script>

---

## 参考

---

- [Using CSS Variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_variables)

- [CSS Variables: Why Should You Care?](https://developers.google.com/web/updates/2016/02/css-variables-why-should-you-care)