## 框架中的网格布局

在框架中使用网格系统(Bootstrap)：

    :::Html
    <!-- Stack the columns on mobile by making one full-width and the other half-width -->
    <div class="row">
      <div class="col-xs-12 col-md-8">.col-xs-12 .col-md-8</div>
      <div class="col-xs-6 col-md-4">.col-xs-6 .col-md-4</div>
    </div>

问题：

+ 基于特定框架，也就是必须要使用框架才能使用网格布局

+ 基于文档结构 + 类名而不是由 CSS 来控制布局

## 使用 CSS 模拟网格布局

### 浮动布局

+ [固定宽度的网格](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Grids#A_simple_fixed_width_grid)。问题是宽度固定，不能随设备尺寸改变而自适应。而且必须计算每个元素所占宽度。

+ [流式网格](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Grids#Creating_a_fluid_grid)。使用百分比来进行表示，同样需要大量计算。

### 弹性盒子？

弹性盒子的问题也是一个一维的布局，是用来处理一条线上（行或者列）的布局的。

## CSS 原生网格

CSS 即将可以原生支持网格布局，目前版本的 Chrome(56) 和 Firefox(51)默认还未支持，接下来的版本即将支持。完整的支持列表可以看[这里](http://caniuse.com/#feat=css-grid)。

### 相关学习资料

+ MDN 网格布局简介： [Grids](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Grids#Updating_our_grid)

+ MDN 网格布局 CSS 参考： [CSS Grid Layout](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Grid_Layout)

+ 网格布局实例： [Grid by Example](http://gridbyexample.com/examples/)

+ CSS TRICKS： [A Complete Guide to Grid](https://css-tricks.com/snippets/css/complete-guide-grid/)

+ W3C官方标准: [CSS Grid Layout Module Level 1](https://www.w3.org/TR/css-grid-1/#grid-line-concept)