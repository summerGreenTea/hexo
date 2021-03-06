title: 为什么要把JS放在body底部？
date: 2016-01-24 19:00:26
tags:
- javascript
---
在前端的"启蒙"时期就在Yahoo军规里面看到关于网页性能优化的这条军规，在实际的代码书写中，我也一直遵循着这条规则。但是在很长一段时间里，我却不知其真正所以然。每次看到这些页面优化法则的时候，附带的解释总是那么几句，心里总是要要打一个大大的问号。这篇博客，就是梳理我对这个问题的一些了解和思考。
<!--more-->

## 前言
前些天，在微博上看到了一篇关注度很高的文章[JS 一定要放在 Body 的最底部么？聊聊浏览器的渲染机制](http://web.jobbole.com/84843/),让我联想到一系列在之前接触了解到的浏览器渲染机制和Javascript单线程的知识，刚好借看到的这篇文章的机会做个详细的梳理。

## 浏览器的渲染机制
我经常会看到一类面试题就是__一个页面，从输入URL地址到页面被加载出来都发生了什么?__,事实上，关于这个问题，我觉得往小的说，其实就是浏览器的渲染机制，要从大的方面来说，还涉及到DNS寻址，http、TCP/IP协议、路由选择等计算机网络相关方面的知识，具体这里有一篇文章，讲的相当详细，[从输入 URL 到页面加载完成的过程中都发生了什么事情？](http://web.jobbole.com/83720/)，自认为自己还没达到能说破天际的水平，所以还是往小的来讲。

先来看下浏览器的主要结构:
{% asset_img layers.png %}
User Interface : 包括地址栏，前进后退，书签菜单等窗口上除了网页显示区域以外的部分。
Boswer engine : 查询与操作渲染引擎的接口。
Rendering engine : 负责显示请求的内容。比如请求到HTML, 它会负责解析HTML 与 CSS 并将结果显示到窗口中。

	* Networking ：用于网络请求, 如HTTP请求。它包括平台无关的接口和各平台独立的实现。
	* Javascript Interpreter ：用于解析执行JavaScript代码。
	* Ui Backend : 绘制基础元件，如组合框与窗口。它提供平台无关的接口，内部使用操作系统的相应实现。

这里借一张webkit的main flow图片来开个头：
{% asset_img webkitflow.png %}
当服务器响应我们的请求并发回我们想要访问页面的HTML代码的时候，浏览器的work flow是:

* 首先调用__Rendering engine__的__Ui Backend__去解析获得的HTML代码，这个时候浏览器如果发现HTML中有外部资源(比如css、js以及img)，就会调用__Networking__并行向资源所在服务器发起HTTP请求来请求返回资源。当页面中所有的HTML代码都被解析完成后，就会生成__DOM Tree__。这是一个树形的数据结构，而且DOM Tree的构建过程是一个深度遍历的过程。利用代码和图片表示下就为:

```
<html>
	<body>
		<p>
			Hello World
		</p>
		<div> <img src="example.png"/></div>
	</body>
</html>
```
将被转化为:
{% asset_img dom-tree.png %}

* 浏览器将请求的每个CSS文件下载好并生成一个__StyleSheet object__，每个对象都包含若干个__CSS Rule__，这些CSS Rule对象包含__Selector__和__Declaration object__，其他的对象就是CSS的语法。

{% asset_img css-rule.png %}

* 浏览器会将生成的DOM Tree和CSS Rule结合形成__Render Tree__。

{% asset_img dom-tree-and-css-rules.jpg %}

* 接下浏览器要做的就是将生成的Render Tree渲染到我们的用户界面当中。这一步就是__Layout__。浏览器会计算每个DOM节点上的位置信息，来确定节点在用户界面的位置(这个过程操作相对复杂、耗时，也是DOM操作缓慢的原因之一)。

* 当浏览器知道相应节点的位置信息后，渲染引擎将Render Tree中的节点中的其他渲染规则应用到节点上。

当结束前面的步骤后，一个HTML页面就会被渲染出来。似乎已经全部讲完了，但整个过程却还没有那么简单的结束哦！

## Ui Backend阻塞 
我们在第一步的时候提到了，渲染引擎会在解析到有外部资源的HTML代码时，调用网络去发起HTTP请求。当请求的是Js文件并且Js文件下载完成后，会将Ui Backend(我理解为UI Thread)阻塞，并等待JS代码下载解析完成才唤醒Ui Backend。

这个是因为Js不仅可以完成逻辑操作，还可以对DOM的操作来改变页面的内容，如果Ui Backend继续渲染，那么等到渲染完成后再执行Js,而这时的Js中有大量对DOM的操作的话，那么势必会造成大量的Repaint/Reflow。这种做法显然不太合适，所以浏览器选择去执行Rendering engine 中的Javascript Interpreter(我理解为Js Thread)，而阻塞Ui Backend，显然，用户需要等待更长的时间才能看到他们想要访问的页面。

出于对这种情况的考虑，所以很多人建议把JS文件放在Body底部，这样就可以等到页面渲染完成后再执行JS。

## 思考

通过上面内容的了解，我们可以得出将JS文件放在页面底部的做法是非常可取的结论，那么在实际的工程项目运用是是否真的是如此呢？答案是否定的。

思考一个应用场景，我们通常需要根据用户的浏览器版本以及用户打开页面第一时间的信息(当前的网络环境等等)来记录一些数据或处理一些兼容性问题。常见的有_html5shim.js_等类库，需要处理兼容低版本浏览器，但是如果等到页面加载完成之后再加载这类文件的话，就会造成一些不知名的错误。但是通常来说，具体的项目环境，会有不同的使用规则。知乎上这个问题的[金戈铁马的回答](https://www.zhihu.com/question/34147508)就很能说明这个问题。

当然这是特殊的情况都有一个共同点，就是这些JS文件必须不能涉及到页面逻辑，也就是不能存在操作DOM代码。



