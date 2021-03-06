---
title: 浏览器页面渲染流程梳理
date: 2018-05-23T21:53:54+08:00
categories: ["JavaScript"]
tags : ["JavaScript", "Browser"]
toc: true
# featured_image : ""
keywords: ["Browser", "浏览器渲染"]
description : "浏览器页面渲染流程梳理"
---


## 浏览器渲染基本流程
浏览器渲染流程如下图所示：

![](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-webkitflow.png)


大概可以划分成以下几个步骤：

1. 通过HTML解析器解析HTML文本并构建DOM tree
2. 通过CSS解析器解析CSS样式表并构建CSSOM tree
3. 根据DOM tree 和 CSSOM tree 构建 Render tree
4. Render tree 刚构建完后是没有元素节点坐标、尺寸大小等信息的，此时需要通过Layout(Reflow)进行布局处理，计算出元素在屏幕上显示的位置，尺寸大小等信息。
5. 遍历渲染树，对每一个元素节点进行绘制（Painting）


### 解析（Parsing）
解析的过程分为两个步骤：词法分析和语法分析。
词法分析负责将输入内容分解成一个个有效标记；而语法分析负责根据语言的语法规则分析文档的结构，从而构建解析树。通过词法分析可以将无关的字符（比如空格和换行符）分离出来。


<!-- ![](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-parsing.png) -->


解析是一个迭代的过程。通常，解析器会向词法分析器请求一个新标记，并尝试将其与某条语法规则进行匹配。如果发现了匹配规则，解析器会将一个对应于该标记的节点添加到解析树中，然后继续请求下一个标记。

如果没有规则可以匹配，解析器就会将标记存储到内部，并继续请求标记，直至找到可与所有内部存储的标记匹配的规则。如果找不到任何匹配规则，解析器就会引发一个异常。这意味着文档无效，包含语法错误。


### 转译(Translation)
很多时候，解析树还不是最终产品。解析通常是在转译过程中使用的，而转译是指将输入文档转换成另一种格式。编译就是这样一个例子。编译器可将源代码编译成机器代码，具体过程是首先将源代码解析成解析树，然后将解析树翻译成机器代码文档。

<!-- ![](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-translate.png) -->

### HTML解析
解析器的输出“解析树”是由 DOM 元素和属性节点构成的树结构。DOM 是文档对象模型 (Document Object Model) 的缩写。它是 HTML 文档的对象表示，同时也是外部内容（例如 JavaScript）与 HTML 元素之间的接口。
解析树的根节点是“[Document](https://www.w3.org/TR/1998/REC-DOM-Level-1-19981001/level-one-core.html#i-Document)”对象。

DOM 与标记之间几乎是一一对应的关系。比如下面这段标记：
```javascript
<html>
  <body>
    <p>
      Hello World
    </p>
    <div> <img src="example.png"/></div>
  </body>
</html>
```
可翻译成如下的 DOM 树：


![示例标记的 DOM 树](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-domtree.png)



##### 解析算法

[HTML5 规范详细地描述了解析算法](https://html.spec.whatwg.org/multipage/parsing.html)。此算法由两个阶段组成：标记化和树构建。

标记化是词法分析过程，将输入内容解析成多个标记。HTML 标记包括起始标记、结束标记、属性名称和属性值。

标记生成器识别标记，传递给树构造器，然后接受下一个字符以识别下一个标记；如此反复直到输入的结束。


![HTML 解析流程](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-htmlparse.png)



### CSS解析
和 HTML 不同，CSS 是上下文无关的语法。事实上，[CSS 规范定义了 CSS 的词法和语法](https://www.w3.org/TR/CSS2/grammar.html)。

WebKit 使用 Flex 和 Bison 解析器生成器，通过 CSS 语法文件自动创建解析器。Bison 会创建自下而上的移位归约解析器。Firefox 使用的是人工编写的自上而下的解析器。这两种解析器都会将 CSS 文件解析成 StyleSheet 对象，且每个对象都包含 CSS 规则。CSS 规则对象则包含选择器和声明对象，以及其他与 CSS 语法对应的对象。

![解析 CSS](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-cssparse.png)



### 处理脚本和样式表的顺序

#### 脚本

网络的模型是同步的。网页解析器遇到 `<script>` 标记时文档的解析将停止，直到脚本执行完毕。如果脚本是外部的，那么解析过程会停止，直到从网络同步抓取资源完成后再继续。你可以在`<script>` 标签上添加“defer”属性（`<script defer>`），这样它就不会停止文档解析，而是等到解析结束才执行。HTML5 增加了一个async属性，可将脚本标记为异步`<script async>`），以便由其他线程解析和执行。

#### 预解析
WebKit 和 Firefox 都进行了这项优化。在执行脚本时，其他线程会解析文档的其余部分，找出并加载需要通过网络加载的其他资源。通过这种方式，资源可以在并行连接上加载，从而提高总体速度。请注意，预解析器不会修改 DOM 树，而是将这项工作交由主解析器处理；预解析器只会解析外部资源（例如外部脚本、样式表和图片）的引用。

#### 样式表
另一方面，样式表有着不同的模型。理论上来说，应用样式表不会更改 DOM 树，因此似乎没有必要等待样式表并停止文档解析。但这涉及到一个问题，就是脚本在文档解析阶段会请求样式信息。如果当时还没有加载和解析样式，脚本就会获得错误的回复，这样显然会产生很多问题。这看上去是一个非典型案例，但事实上非常普遍。Firefox 在样式表加载和解析的过程中，会禁止所有脚本。而对于 WebKit 而言，仅当脚本尝试访问的样式属性可能受尚未加载的样式表影响时，它才会禁止该脚本。


### Render tree构建

![Render tree构建](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/render-tree-construction.png)


Render tree是由 DOM 和 CSSOM 组合构建而成的。也是页面可视化元素按照其显示顺序而组成的树，是文档的可视化表示。它的作用是让浏览器按照正确的顺序绘制内容。

Firefox 将Render tree中的元素称为“框架”。WebKit 使用的术语是呈现器或呈现对象。
呈现器知道如何布局并将自身及其子元素绘制出来。


#### 呈现树和 DOM 树的关系
呈现器是和 DOM 元素相对应的，但并非一一对应。非可视化的 DOM 元素不会插入呈现树中，例如“head”元素。如果元素的 display 属性值为“none”，那么也不会显示在呈现树中（但是 visibility 属性值为“hidden”的元素仍会显示）。
有一些 DOM 元素对应多个可视化对象。它们往往是具有复杂结构的元素，无法用单一的矩形来描述。例如，“select”元素有 3 个呈现器：一个用于显示区域，一个用于下拉列表框，还有一个用于按钮。如果由于宽度不够，文本无法在一行中显示而分为多行，那么新的行也会作为新的呈现器而添加。
另一个关于多呈现器的例子是格式无效的 HTML。根据 CSS 规范，inline 元素只能包含 block 元素或 inline 元素中的一种。如果出现了混合内容，则应创建匿名的 block 呈现器，以包裹 inline 元素。

有一些呈现对象对应于 DOM 节点，但在树中所在的位置与 DOM 节点不同。浮动定位和绝对定位的元素就是这样，它们处于正常的流程之外，放置在树中的其他地方，并映射到真正的框架，而放在原位的是占位框架。

![呈现树及其对应的 DOM 树](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-render&dom.png)

### 布局（Layout/Reflow）
当Render Tree刚构建完时，并不包含元素节点的位置和大小信息。计算这些值的过程称为布局或重排。

HTML 采用基于流的布局模型，这意味着大多数情况下只要一次遍历就能计算出几何信息。处于流中靠后位置元素通常不会影响靠前位置元素的几何特征，因此布局可以按从左至右、从上至下的顺序遍历文档。但是也有例外情况，比如 HTML 表格的计算就需要不止一次的遍历。

坐标系是相对于根框架而建立的，使用的是上坐标和左坐标。

布局是一个递归的过程。它从根呈现器（对应于 HTML 文档的 <html> 元素）开始，然后递归遍历部分或所有的框架层次结构，为每一个需要计算的呈现器计算几何信息。

根呈现器的位置左边是 0,0，其尺寸为视口（也就是浏览器窗口的可见区域）。
所有的呈现器都有一个“layout”或者“reflow”方法，每一个呈现器都会调用其需要进行布局的子代的 layout 方法。

#### Dirty 位系统
为避免对所有细小更改都进行整体布局，浏览器采用了一种“dirty 位”系统。如果某个呈现器发生了更改，或者将自身及其子代标注为“dirty”，则需要进行布局。

有两种标记：“dirty”和“children are dirty”。“children are dirty”表示尽管呈现器自身没有变化，但它至少有一个子代需要布局。

#### 全局布局和增量布局
全局布局是指触发了整个呈现树范围的布局，触发原因可能包括：

1. 影响所有呈现器的全局样式更改，例如字体大小更改。
2. 屏幕大小调整。

布局可以采用增量方式，也就是只对 dirty 呈现器进行布局（这样可能存在需要进行额外布局的弊端）。
当呈现器为 dirty 时，会异步触发增量布局。例如，当来自网络的额外内容添加到 DOM 树之后，新的呈现器附加到了呈现树中。


![增量布局](https://img-1256541035.cos.ap-shanghai.myqcloud.com/imgs/browserRendering-reflow.png)

*图：增量布局 - 只有 dirty 呈现器及其子代进行布局*

### 绘制
在绘制阶段，系统会遍历呈现树，并调用呈现器的“paint”方法，将呈现器的内容显示在屏幕上。绘制工作是使用用户界面基础组件完成的。

#### 全局绘制和增量绘制
和布局一样，绘制也分为全局（绘制整个呈现树）和增量两种。在增量绘制中，部分呈现器发生了更改，但是不会影响整个树。更改后的呈现器将其在屏幕上对应的矩形区域设为无效，这导致 OS 将其视为一块“dirty 区域”，并生成“paint”事件。OS 会很巧妙地将多个区域合并成一个。在 Chrome 浏览器中，情况要更复杂一些，因为 Chrome 浏览器的呈现器不在主进程上。Chrome 浏览器会在某种程度上模拟 OS 的行为。展示层会侦听这些事件，并将消息委托给呈现根节点。然后遍历呈现树，直到找到相关的呈现器，该呈现器会重新绘制自己（通常也包括其子代）。

#### 绘制顺序

[CSS2 规范定义了绘制流程的顺序](https://www.w3.org/TR/CSS21/zindex.html)。绘制的顺序其实就是元素进入堆栈样式上下文的顺序。这些堆栈会从后往前绘制，因此这样的顺序会影响绘制。块呈现器的堆栈顺序如下：
1. 背景颜色
2. 背景图片
3. 边框
4. 子代
5. 轮廓

#### WebKit 矩形存储
在重新绘制之前，WebKit 会将原来的矩形另存为一张位图(Bitmap)，然后只绘制新旧矩形之间的差异部分。
#### 动态变化
在发生变化时，浏览器会尽可能做出最小的响应。因此，元素的颜色改变后，只会对该元素进行重绘。元素的位置改变后，只会对该元素及其子元素（可能还有同级元素）进行布局和重绘。添加 DOM 节点后，会对该节点进行布局和重绘。一些重大变化（例如增大“html”元素的字体）会导致缓存无效，使得整个呈现树都会进行重新布局和绘制。



参考链接
* https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/
* https://www.youtube.com/watch?v=SmE4OwHztCc
* https://www.youtube.com/watch?v=0IsQqJ7pwhw