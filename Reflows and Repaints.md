#### 浏览器渲染机制

1. 解析代码：HTML 代码解析为 DOM，CSS 代码解析为 CSSOM（CSS Object Model）。
2. 对象合成：将 DOM 和 CSSOM 合成一棵渲染树（render tree）。
3. 布局：计算出渲染树的布局（layout）。
4. 绘制：将渲染树绘制到屏幕。

以上四步并非严格按顺序执行，往往第一步还没完成，第二步和第三步就已经开始了。所以，会看到这种情况：网页的 HTML 代码还没下载完，但浏览器已经显示出内容了。

#### 重绘和重流（Repaints and reflows）

渲染树转换为网页布局，称为“布局流”（flow）；布局显示到页面的这个过程，称为“绘制”（paint）。它们都具有阻塞效应，并且会耗费很多时间和计算资源。

一个页面初始化的时候至少伴随着一次布局和绘制（除非你的页面什么都没有）。之后，改变用于构建渲染树的输入信息会导致重流或者重绘，或者两者都会改变：

1. DOM元素的位置、大小发生改变，即网页的布局发生了改变，浏览器根据计算结果重新将其放到它该有的位置，这就叫做**重流**
2. DOM元素的位置、大小没有发生改变，有可能是因为节点的几何属性发生了变化，或者是因为样式发生了变化，比如说改变了背景颜色。这个屏幕更新称为**重绘**

重绘和重流的代价很高，会损害用户体验，会消耗很多时间和计算资源，使UI界面迟缓，即页面加载慢、卡顿。**重流和重绘并不一定一起发生，重流必然导致重绘，重绘不一定需要重流**

#### 什么会导致重流或者重绘呢？

任何改变用于构建渲染树的输入信息的操作都能导致重流或者重绘，例如：

- 添加、移除、更新DOM节点
- 使用`display: none`来隐藏一个DOM节点会导致重流和重绘，`visibility: hidden`则只会造成重绘，因为没有几何改变。
- DOM节点在页面上移动、做动画效果时
- 添加样式表，调整样式属性
- 用户操作如：设置了鼠标悬停（`a:hover`）效果、页面滚动、在输入框中输入文本、改变窗口大小等等。

```javascript
var bstyle = document.body.style; // cache
 
bstyle.padding = "20px"; // reflow, repaint
bstyle.border = "10px solid red"; // another reflow and a repaint
 
bstyle.color = "blue"; // repaint only, no dimensions changed
bstyle.backgroundColor = "#fad"; //repaint
 
bstyle.fontSize = "2em"; // reflow, repaint
 
// new DOM element - reflow, repaint
document.body.appendChild(document.createTextNode('dude!'));
```

有些回流可能比其他回流更昂贵。想象一下渲染树——如果在树的下面摆弄一个节点（主体的一个直接后代），那么可能不会让许多其他节点失效。但是，当在页面顶部生成一个div，导致页面上的其他部分全部往下移，这听起来代价就十分昂贵了。

#### 浏览器是智能的

由于与渲染树更改关联的重流和重绘非常昂贵，所以浏览器的目标是减少负面影响。一种策略是干脆不做这项工作。或者至少不是现在。浏览器将设置脚本所需更改的队列，并分批执行。通过这种方式，将组合每个更改都需要重新流的几个更改，并且只计算一个重新流。浏览器可以将更改添加到队列中，然后在经过一定时间或达到一定数量的更改后刷新队列（**类似节流函数**）。

但有时脚本可能会阻止浏览器优化reflows，并导致它刷新队列并执行所有批量更改。这会发生在请求样式信息时，例如

1. `offsetTop`, `offsetLeft`, `offsetWidth`, `offsetHeight`
2. `scrollTop`、`scrollLeft`、`scrollWidth`、`scrollHeight`
3. `clientTop`、`clientLeft`、`clientWidth`、`clientHeight`
4. `getComputedStyle()`, or `currentStyle` in IE

以上所有这些本质上都是在请求关于节点的样式信息，并且无论何时执行此操作，浏览器都必须提供最新的值。为了做到这一点，它需要应用所有更改，刷新队列，重流。

例如，快速连续地设置和获取样式(在循环中)不是一个好主意，像:

`el.style.left = el.offsetLeft + 10 + "px";`

#### 最小化重绘和重流

作为开发者，应该尽量设法降低重绘的次数和成本。比如，尽量不要变动高层的 DOM 元素，而以底层 DOM 元素的变动代替；再比如，重绘`table`布局和`flex`布局，开销都会比较大。

- 不要一条一条的去修改样式，为了完整性和可维护性，在静态样式中应该去更改类名而不是样式。若样式是动态的，则编辑`cssText`属性而不是对元素及其样式属性进行每一点更改。

  ```javascript
  //不好的写法
  var left = 10, top = 10;
  el.style.left = left + "px";
  el.style.top = top + "px";
  
  //更好的写法,先预定好css的class,后修改DOM的className
  <style>
      .theclassname {
          left: 10px;
          top : 10px;
      }
  </style>
  el.className += "theclassname"
  ```

- 批量DOM更改并“脱机”执行:
  - 使用`documentFragment`来保存临时更改
  - 克隆要更新的节点，处理克隆副本，然后用更新的克隆交换原始节点
  - 先设置`DOM`的`display`为`none` (`display: none`,此时有一次repaint)，然后想怎么修改这个DOM都可以，比如修改个100次，最后再把`display`属性设置回来，显现DOM节点（这样，虽然操作了100次，但是只有两次reflows）

- 不要把 `DOM` 节点的属性值放在一个循环里当成循环里的变量。不然这会导致大量地读写这个结点的属性。如果需要处理一个计算值，将其缓存到一个本地变量并处理本地副本。

   ```javascript
  //不好的写法,DOM节点属性值每循环一次就要修改一次，就要reflow一次
  for(big;loop;here){
      el.style.left = el.offsetLeft + 10 + "px";
      el.style.top = el.offsetTop + 10 + "px";
  }
  
  //更好的写法
  var left = el.offsetLeft,
      top = el.offsetTop,
      esty = el.style;
  for(big;loop;here){
      left += 10;
      top + = 10;
      esty.left = left + "px";
      esty.top = top + "px";
  }
   ```

- 一般来说，考虑一下渲染树，以及更改后需要重新渲染部分的大小。例如，为动画的 `HTML` 元件使用 `fixed` 或 `absoult` 的 `position`，那么修改他们的 `CSS` 是会大大减小 `reflow` 。

- 使用`window.requestAnimationFrame()`，因为它可以把代码推迟到下一次重流时执行，而不是立即要求页面重流。

   ```javascript
  // 重绘代价高
  function doubleHeight(element) {
    var currentHeight = element.clientHeight;
    element.style.height = (currentHeight * 2) + 'px';
  }
  
  all_my_elements.forEach(doubleHeight);
  
  // 重绘代价低
  function doubleHeight(element) {
    var currentHeight = element.clientHeight;
  
   //使用了window.requestAnimationFrame()
    window.requestAnimationFrame(function () {
      element.style.height = (currentHeight * 2) + 'px';
    });
  }
  
  all_my_elements.forEach(doubleHeight);
  
  //上面的第一段代码，每读一次 DOM，就写入新的值，会造成不停的重排和重流。第二段代码把所有的写操作，都累积在一起，从而 DOM 代码变动的代价就最小化了。
   ```



参考链接：

[1]  [阮一峰 : JavaScript标准参考教程](http://javascript.ruanyifeng.com/bom/engine.html)

[2] [Rendering: repaint, reflow/relayout, restyle](http://www.phpied.com/rendering-repaint-reflowrelayout-restyle/)