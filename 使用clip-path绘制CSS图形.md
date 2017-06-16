一直以来使用CSS绘制图形都十分麻烦，比如为了绘制一个三角形，常用的方法是通过控制`border`来进行绘制，圆形则通过`border-radius`绘制。这些方法虽然也能达到效果，但一来十分复杂，二来绘制的过程完全不符合人类思考方式。原因就在于这些属性在设计之初就不是用作这些用途的。`clip-path`属性就是专门用来处理这类问题的，目前主流浏览器已实现对其基础支持。

## clip-path 基本用法

### 绘制三角形

#### HTML

    :::html
    <div class="clip demo1"></div>


#### CSS

    :::css
    .clip {
        width: 200px;
        height: 200px;
    }
    .demo1 {
        background-color: #1ce4c2;
        clip-path: polygon(50% 0, 100% 100%, 0 100%);
    }


<div class="clip demo1"></div>
<style>
    .clip {
        width: 200px;
        height: 200px;
    }
    .demo1 {
        background-color: #1ce4c2;
        clip-path: polygon(50% 0, 100% 100%, 0 100%);
    }
</style>

### 绘制圆形

#### HTML

    :::html
    <div class="clip demo2"></div>


#### CSS

    :::css
    .demo2 {
        background-color: #f2c145;
        clip-path: polygon(50% 0, 100% 100%, 0 100%);
    }


<div class="clip demo2"></div>
<style>
    .demo2 {
        background-color: #f2c145;
        clip-path: circle();
    }
</style>

### 绘制多边形

#### HTML

    :::html
    <div class="clip demo3"></div>


#### CSS

    :::css
    .demo3 {
        background-color: #cf45c8;
        clip-path: polygon(20% 0, 80% 0, 100% 20%, 100% 80%, 80% 100%, 20%  100%, 0 80%, 0 20%);
    }


<div class="clip demo3"></div>
<style>
    .demo3 {
        background-color: #cf45c8;
        clip-path: polygon(20% 0, 80% 0, 100% 20%, 100% 80%, 80% 100%, 20% 100%, 0 80%, 0 20%);
    }
</style>

### 绘制椭圆


#### HTML

    :::html
    <div class="clip demo4"></div>


#### CSS

    :::css
    .demo4 {
        background-color: #23ef6a;
        clip-path: ellipse(40% 25% at 50% 50%);
    }


<div class="clip demo4"></div>
<style>
    .demo4 {
        background-color: #23ef6a;
        clip-path: ellipse(40% 25% at 50% 50%);
    }
</style>

## 动画效果

### 动画一

#### HTML

    :::html
    <div class="clip demo4"></div>


#### CSS

    :::css
    .demo4 {
        background-color: #cf45c8;
        clip-path: polygon(20% 0, 80% 0, 100% 20%, 100% 80%, 80% 100%,  20%100%, 0 80%, 0 20%);
        animation: circle 1s infinite alternate;
    }
    @keyframes circle {
        0% {
            clip-path: circle(100px)
        }
        100% {
            clip-path: circle(0)
        }
    }


<div class="clip demo5"></div>
<style>
    .demo5 {
        background-color: #cf45c8;
        clip-path: polygon(20% 0, 80% 0, 100% 20%, 100% 80%, 80% 100%, 20% 100%, 0 80%, 0 20%);
        animation: circle 1s infinite alternate;
    }
    @keyframes circle {
        0% {
            clip-path: circle(100px)
        }
        100% {
            clip-path: circle(0)
        }
    }
</style>

### 动画二

#### HTML

    :::html
    <div class="clip demo6"></div>


#### CSS

    :::css
    .demo6 {
        background-color: #8ac351;
        clip-path: polygon(20% 0, 80% 0, 100% 20%, 100% 80%, 80% 100%,  20%100%, 0 80%, 0 20%);
        animation: msg 1s infinite alternate ease-in-out;
    }
    @keyframes msg {
        0% {
            clip-path: polygon(0% 0%, 100% 0%, 100% 75%, 75% 75%, 75% 100%, 50% 75%, 0% 75%)
        }
        100% {
            clip-path: polygon(50% 0%, 90% 20%, 100% 60%, 75% 100%, 25% 100%,   0% 60%, 10% 20%)
        }
    }


<div class="clip demo6"></div>
<style>
    .demo6 {
        background-color: #8ac351;
        clip-path: polygon(20% 0, 80% 0, 100% 20%, 100% 80%, 80% 100%, 20% 100%, 0 80%, 0 20%);
        animation: msg 1s infinite alternate ease-in-out;
    }
    @keyframes msg {
        0% {
            clip-path: polygon(0% 0%, 100% 0%, 100% 75%, 75% 75%, 75% 100%, 50% 75%, 0% 75%)
        }
        100% {
            clip-path: polygon(50% 0%, 90% 20%, 100% 60%, 75% 100%, 25% 100%, 0% 60%, 10% 20%)
        }
    }
</style>

## 制作有趣的效果

### HTML

    :::html
    <div id="dark">
        <div id="ring">
            <h1>The Ring</h1><p>Three Rings for the Elven-kings under the   sky,
            Seven for the Dwarf-lords in their halls of stone,
            Nine for Mortal Men doomed to die,
            One for the Dark Lord on his dark throne
            In the Land of Mordor where the Shadows lie.
            One Ring to rule them all, One Ring to find them,
            One Ring to bring them all, and in the darkness bind them,
            In the Land of Mordor where the Shadows lie。</p>
        </div>
    </div>

### CSS

    :::css
    #dark {
        background-color: #000;
    }
    #ring{
        background: #fff;
        padding: 1em;
        font-family: 'Times New Roman', Times, serif;
        text-align: center;
        white-space: pre;
        clip-path: circle(50px at 200px 100px)
    }


### JS

    :::js
    let ring = document.querySelector('#ring')
    window.addEventListener('mousemove', (e) => {
        let x = e.clientX
        let y = e.clientY
        let ox = ring.offsetLeft - document.body.scrollLeft
        let oy = ring.offsetTop - document.body.scrollTop
        let newPath = `circle(50px at ${x-ox}px ${y-oy}px)`
        ring.style['clip-path'] = newPath
    })

### 在下图移动鼠标查看效果

<div id="dark">
<div id="ring">
    <h1>The Ring</h1><p>Three Rings for the Elven-kings under the sky,
Seven for the Dwarf-lords in their halls of stone,
Nine for Mortal Men doomed to die,
One for the Dark Lord on his dark throne
In the Land of Mordor where the Shadows lie.
One Ring to rule them all, One Ring to find them,
One Ring to bring them all, and in the darkness bind them,
In the Land of Mordor where the Shadows lie。</p>
</div>
</div>
<style>
#dark {
    background-color: #000
}
#ring{
    background: #fff;
    font-family: 'Times New Roman', Times, serif;
    text-align: center;
    white-space: pre;
    clip-path: circle(50px at 200px 100px)
}
</style>
<script>
let ring = document.querySelector('#ring')
window.addEventListener('mousemove', (e) => {
    let x = e.clientX
    let y = e.clientY
    let ox = ring.offsetLeft - document.body.scrollLeft
    let oy = ring.offsetTop - document.body.scrollTop
    let newPath = `circle(50px at ${x-ox}px ${y-oy}px)`
    ring.style['clip-path'] = newPath
})
</script>

## 配合shape-outsize进行排版

`shape-outsized`属性与`clip-path`相似，都可以剪切除一定的图形。不过`shape-outsized`主要与`float`属性配合使用，使文字围绕浮动元素剪切过的路径而不是浮动元素的边缘。

    :::html
    <div>
        <div id="float"></div>
        <p id="p">Three Rings for the Elven-kings under the sky,
            Seven for the Dwarf-lords in their halls of stone,
            Nine for Mortal Men doomed to die,
            One for the Dark Lord on his dark throne
            In the Land of Mordor where the Shadows lie.
            One Ring to rule them all, One Ring to find them,
            One Ring to bring them all, and in the darkness bind them,
            In the Land of Mordor where the Shadows lie。
        </p>
    </div>

### CSS

    :::css
    #float {
        width: 200px;
        height: 200px;
        background-color: #23bcc7;
        clip-path: polygon(50% 0%, 61% 35%, 98% 35%, 68% 57%, 79% 91%, 50% 70%, 21% 91%,    32% 57%, 2% 35%, 39% 35%);
        shape-outside: polygon(50% 0%, 61% 35%, 98% 35%, 68% 57%, 79% 91%, 50% 70%, 21%     91%, 32% 57%, 2% 35%, 39% 35%);
        float: left;
    }

<div>
<div id="float"></div>
<p id="p">Three Rings for the Elven-kings under the sky,
Seven for the Dwarf-lords in their halls of stone,
Nine for Mortal Men doomed to die,
One for the Dark Lord on his dark throne
In the Land of Mordor where the Shadows lie.
One Ring to rule them all, One Ring to find them,
One Ring to bring them all, and in the darkness bind them,
In the Land of Mordor where the Shadows lie。</p>
</div>
<style>
#float {
    width: 150px;
    height: 150px;
    background-color: #23bcc7;
    clip-path: polygon(50% 0%, 61% 35%, 98% 35%, 68% 57%, 79% 91%, 50% 70%, 21% 91%, 32% 57%, 2% 35%, 39% 35%);
    shape-outside: polygon(50% 0%, 61% 35%, 98% 35%, 68% 57%, 79% 91%, 50% 70%, 21% 91%, 32% 57%, 2% 35%, 39% 35%);
    float: left;
}
</style>

---

## 参考

- [clip-path](https://developer.mozilla.org/en-US/docs/Web/CSS/clip-path)

- [shape-outside](https://developer.mozilla.org/en-US/docs/Web/CSS/shape-outside)