# DOM编程


## 浏览器中的DOM

文档对象模型(DOM)是一个独立于语言的，用于操作XML和HTML文档的程序接口(API)。尽管DOM是个与语言无关的API，它在浏览器中的接口却是用JavaScript实现的。客户端脚本编程大多数时候是和底层文档打交道，DOM就成为现在JavaScript编码中的重要部分。


**天生就慢**

浏览器通常会把DOM和JavaScript独立实现。而两个相互独立的功能只能通过接口彼此相连，就会产生消耗。一个贴切比喻，将DOM和JavaScript(这里指ECMAScript)比喻成两个岛，它们之间用收费桥梁连接，每次ECMAScirpt访问DOM都要过桥交过桥费，因此访问DOM次数越多，费用就越高。推荐做法是尽可能减少过桥次数，努力待在ECMAScript岛上。


## DOM访问与修改

上面提到，访问DOM元素要交“过桥费”，而修改元素则更昂贵，因为它会导致浏览器重新计算页面的几何变化，最坏的情况下就是在循环中访问和修改元素，尤其是对HMTL元素几何循环操作。如：

```javascript
function innerHTMLLoop() {
	for(var count = 0; count < 15000; count++) {
		document.getElementById('here').innerHTML += 'a';
	}
}

function innerHTMLLoop2() {
	var content = '';
	for(var count = 0;count < 15000; count++) {
		coutent += 'a';
	}
	document.getElementById('here').innerHTML += content;
}
```

使用innerHTMLLoop2()比使用innerHTMLLoop()快155倍。

因此，通用的经验法则：**减少访问DOM的次数，把运算尽量留在JavaScript处理。**

### innerHTML对比DOM方法

修改页面区域最佳方案是用非标准但支持良好的innerHTML属性好，还是只用类似document.createElement()的原生DOM方法呢？  ->  相差无几。

### 节点克隆

使用DOM方法更新页面内容的另一个途径是克隆已有元素，而不是创建元素。即使用element.cloneNode()替代document.createElement()。大多数浏览器下，节点克隆更有效率，不过并不是很明显。

### HTML集合

HTML集合是包含了DOM节点引用的类数组对象,并不是真正的数组，但提供了一个length属性，且能以数字索引方式访问元素。

HTML集合一直与文档保持连接，每次需要更新的信息时，都会重复执行查询过程，哪怕仅获取集合长度（length）也是如此，这正是低效之源。

以下方法返回值就是一个集合：

- document.getElementByName()
- document.getElementByClassName()
- document.getElementByTagName()

下面属性同样访问HTML集合:

- document.images
- document.links
- document.forms
- document.forms[0].elements

**昂贵的集合**

```javascript
// 一个意外的死循环（即上面说的每次更新信息都会重复查询）
var alldivs = document.getElementByTagName('div');
for(var i = 0;i < alldivs.length;i++) {
	document.body.appendChild(document.createElement('div'));
}
```

每次创建一个新div并添加到body，alldivs.length都会增加。

修改方案：

```javascript
var coll = document.getElementByTagName('div'),
	len = coll.length;

for(var count = 0; count < len; count++) {
	// 代码处理
}
```

**访问集合元素时使用局部变量**

一般来说，对于任何类型的DOM访问，同一个DOM属性或方法需要多次访问时，最好用一个局部变量缓存此成员。当遍历一个集合时，第一优先原则是把集合存储在局部变量中，并把length缓存在循环外部，使用局部变量替换这些要多次读取的元素。


**获取DOM元素**

通常需要从某个DOM元素开始操作周围的元素，或者递归查找所有子节点，可使用childNodes得到元素集合，或nextSibling获取每个相邻元素。

同样在循环中访问时使用局部变量。


**元素节点**

DOM元素属性诸如childNodes、firstChild和nextSibling并不区分元素节点和其他类型节点，比如注释和文本节点(通常只是两个节点间的空格)，某些情况下只需访问元素节点，因此在循环中可能要检查返回节点的类型并过滤非元素节点，这些类型检查和过滤其实是不必要的DOM操作。

可用的话推荐使用这些API，因为其执行效率高于自己在JavaScript代码中实现过滤。

- children ， 替代childNodes
- childElementCount ， 替换childNodes.length
- firstElementChild ，替换firstChild
- lastElementChild ， 替换lastChild
- nextElementSibling ， 替换nextSibling
- previousElementSibling ，替换previousSibling


**选择器API**

尽量使用更快的API，如querySelectorAll(),querySelectorAll()方法比使用JavaScript和DOM来遍历查找元素要快得多。

```javascript
var elements = document.querySelectorAll('#menu a');
```

该方法不会返回HTML集合，返回的节点也不会对应实时的文档结构，避免了本章讨论的HTML集合引起的性能问题。

### 重绘与重排(Repaints and Reflows)

浏览器下载完页面中所有组件——HMTL标记、JavaScript、CSS、图片——之后解析并生成两个内部数据结构：

- **DOM树**<br/>
表示页面结构
- **渲染树**<br/>
表示DOM节点如何显示

DOM树中的每一个需要显示的节点在渲染树中至少存在一个对应的节点(隐藏的DOM元素在渲染树中没有对应的节点）。一旦DOM和渲染树构建完成，浏览器就开始显示（绘制‘paint’）页面元素。

当DOM的变化影响了元素的几何属性（宽、高）时——如改变边框宽度或给段落增加文字。浏览器需要重新计算元素的几何属性，其他元素的几何属性位置也会受影响，浏览器会使渲染树中受影响的部分失效，并重新构造渲染树。该过程成为“重排”，完成“重排”后，浏览器会重新绘制受影响的部分到屏幕中，该过程称为“重绘”。

**重排何时发生**

- 添加或删除可见的DOM元素
- 元素位置改变
- 元素尺寸改变（外边距、内边距、边框厚度、宽度、高度等属性改变）
- 内容改变，如文本改变或图片被替代为不同尺寸
- 页面渲染器初始化
- 浏览器窗口尺寸改变

根据改变的范围和程度，渲染树中或大或小的对应部分也需要重新计算，有些改变会触发整个页面的重排：滚动条出现

**渲染树变化的排队和刷新**

```javascript
var e = document.getElementById('myDiv');
el.style.borderLeft = '1px';
el.style.borderRight = '2px';
el.style.padding = '5px';
```


上述代码看似执行了三次重排、重绘，但其实只触发了一次，这时因为大多数浏览器通过队列化修改并批量执行(将多个重排过程合并成一次)来优化重排过程。然而，某些操作会强制刷新队列并要求队列中的重排立即执行（这样会使浏览器的优化策略失效）。

比如：

- offsetTop, offsetLeft, offsetWidth, offsetHeight
- scrollTop, scrollLeft, scrollWidth, scrollHeight
- clientTop, clientLeft, clientWidth, clientHeight
- getComputedStyle()(currentStyle in IE)

以上属性和方法需要返回最新布局信息，浏览器不得不执行渲染队列中的‘待处理变化’并触发重排以返回正确值，因此在修改样式过程最好避免上面列出的属性。**尽量不要在布局信息改变时查询它，可以在布局信息改变完毕之后再去查询。**


#### 最小化重绘和重排


**改变样式**


重绘和重排可能代价非常昂贵，因此一个好的提高程序响应速度的策略就是减少此类操作发生，应该合并多次对DOM和样式的修改，一次处理掉。

```javascript
// 不好的写法
var el = document.getElementById('mydiv');
el.style.borderLeft = '1px';
el.style.borderRight = '2px';
el.style.padding = '5px';
```

```
.active {
	border-left-width: 1px;
    border-right-width: 2px;
    padding: 5px;
}
```

```javascript
// 好的写法
var el = document.getElementById('mydiv');
el.classList.add('active');
```

**批量修改DOM**

当需要对DOM元素进行一些列操作时，可以通过以下步骤减少重绘和重排次数。

1. 使文档脱离文档流
2. 对其应用多重改变
3. 把元素带回文档中

有三种基本方法可以使DOM脱离文档：

1. 隐藏元素，应用修改，重新显示
2. 使用文档片断在当前DOM之外构建一个子树，再将其拷贝回文档。
3. 将原始元素拷贝到一个脱离文档的节点中，修改副本，完成后再替换原始元素

例：一个链接列表：

```html
<ul id='fruit'>
  <li> apple </li>
  <li> orange </li>
</ul>
```

若要根据下面对象的附加数据(data)更新上面的这个链接列表：

```javascript
var data = [
	{
		'name': 'Nicholas',
		'url': 'http://nczonline.net'
	},
	{
		'name': 'Ross',
		'url': 'http://techfoolery.net'
	}
]

//用来更新节点数据的通用函数

function appendDataToElement(appendToElement,data) {
	var a,
		li;

	for (var i = 0, max = data.length; i < max; i++) {
		a = document.createElement('a');
		a.href = data[i].url;
		a.appendChild(document.createTextNode(data[i].name));
		li = document.createElement('li');
		li.appendChild(a);
		appendToElement.appenChild(li);
	}
}

var ul = document.getElementById('mylist');
appendDataToElement(ul,data);
```

这种方法，data数组中的每一个新条目被附加到当前DOM树都会导致重排。

**方法一：**通过改变display属性，临时从文档移除&lt;ul>元素，再恢复它。

```javascript
var ul = document.getElementById('mylist');
ul.style.display = 'none';
appendDataToElement(ul,data);
ul.style.display = 'block';
```


**方法二：**在文档外创建并更新一个文档片段，然后将它附加到原始列表中。文档片断时个轻量级的document对象，其设计初衷就是为了完成这类任务——更新和移动节点。它的一个遍历语法是：当你附加一个片段到节点中时，实际上被添加的是该片断地子节点而不是片断本身。

```javascript
// 只触发一次重排，且只访问了一次实时的DOM
var fragment = document.createDocumentFragment();
appendDataToElement(fragment,data);
document.getElementById('mylist').appendChild(fragment);
```


**方法三：**为需要修改的节点创建一个备份，然后对副本进行操作，一旦操作完成，就用新节点代替旧节点。

```javascript
var old = document.getElementById('mylist');
var clone = old.cloneNode(true);
appendDataToElement(clone,data);
old.parentNode.replaceChild(clone,old);
```


**推荐方法二，所产生的DOM遍历和重排次数最少。**



#### 缓存布局信息

当查询布局信息(偏移量、滚动位置、 计算出的样式值)时，浏览器为了返回最新值，会刷新队列并应用所有变更。最好做法是减少布局信息的获取次数，获取后将它赋值给局部变量，再操作局部变量。

#### 让元素脱离动画流

用展开/折叠的方式显示和隐藏部分页面是一种常见的交互模式，它通常是包含展开区域的几何区域，并将页面其他部分推向下方。

一般重排只影响渲染树的一部分，但可能影响很大的部分，甚至整个渲染树，**浏览器所需要重排的次数越少，应用程序的响应速度就越快。**因此页面顶部一个动画推移页面整个余下部分会导致一次代价昂贵的重排，让用户感到页面一顿一顿的。**渲染树需要重新计算的节点越多，情况就越遭。**

使用以下步骤可以避免页面大部分重排：

1. 使用绝对位置定位页面上的动画元素，将其脱离文档流
2. 让元素动起来。当其扩大时，会临时覆盖部分页面。但这只是页面一个小区域的重绘过程，不产生重排或重绘页面的大部分内容。
3. 当动画结束时恢复定位，从而只下移一次文档的其他元素。


## 事件委托

当页面中大量元素都要一次或多次绑定事件处理器时，可能会影响性能。一个简单优雅的DOM事件的技术是事件委托。它基于这样一个事实：事件逐层冒泡并能被父级元素捕获。使用事件代理，只需要给外层元素绑定一个处理器，就可以处理在其子元素上触发的所有事件。

每个事件经历三个阶段：

- 捕获
- 到达目标
- 冒泡

点击子元素，会向上冒泡直到document的顶级乃至window，这使得你可以添加一个事件处理器到父级元素，由它接收所有子节点的事件消息,实现并不复杂，只需要检查事件是否来自所预期的元素。

跨浏览器兼容部分：

- 访问事件对象，并判断事件源
- 取消文档树中的冒泡（可选）
- 阻止默认动作（可选）