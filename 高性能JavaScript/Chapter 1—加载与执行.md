# 加载与执行

JavaScript的阻塞特性令人头疼，即浏览器在执行JavaScript代码时，不能同时做任何事情。多数浏览器使用单一线程处理用户界面（UI）刷新和JavaScript脚本执行，一个时刻只能做一件事。所以当遇到&lt;script>标签时，无论当前JavaScript代码是内嵌还是包含在外链文件，页面的下载和渲染都必须停下来等待脚本执行完成，这是页面生存周期必要环节，因为脚本执行过程可能会修改页面内容。

## 脚本位置

理论上来说，把样式和行为有关的脚本放在一起，并先加载它们，有助于确保页面渲染和交互的正确性。如：

```html
<html>
<head>
    <title>Script Example</title>
    <-- Example of inefficient script positioning -->
    <script type="text/javascript" src="file1.js"></script>
    <script type="text/javascript" src="file2.js"></script>
    <script type="text/javascript" src="file3.js"></script>
	<link rel="stylesheet" type="text/css" href="styles.css">
</head>
<body>
	<p>Hello World!</p>
</body>
</html>
```

这种做法看似正常，但会导致十分严重的性能问题。浏览器在解析到<body>标签之前，不会渲染页面任何部分。<head>中加载了三个JavaScript文件，由于脚本会阻塞页面渲染，直到它们全部下载并执行完成后，页面渲染才会继续。因此将脚本放在页面顶部会导致明显延迟，通常表现为显示空白页面，也无法进行页面交互。


因此，尽量**将脚本放在底部**，<body>标签的底部。

## 组织脚本

每个&lt;script>标签初始下载时都会因执行脚本而导致一定的延时，阻塞页面渲染，减少页面包含的&lt;script>标签(包括内嵌和外链)数量可以最小化延迟时间，改善页面的总性能。

对于外链：考虑到HTTP请求会带来的额外开销，因此下载单个100KB文件会比下载4个25KB文件快，因此减少页面中外链脚本文件的数量将会改善性能。**可以将多个文件合并成一个，以实现仅引用一个&lt;script>标签却加载多个JavaScript文件。**


## 无阻塞的脚本

尽管下载单个较大的JavaScript文件只产生一次HTTP请求，却会锁死浏览器一大段时间，为避免这种情况，需要向页面中逐步加载JavaScript文件。

无阻赛脚本的秘诀在于：在页面加载完成后再加载JavaScript代码，即在window对象的load事件触发之后再下载脚本。实现这种效果的方式：defer属性、async属性。

### 1.延迟的脚本

HTML4为&lt;script>标签定义了一个扩展属性：defer。Defer属性指明脚本不会修改DOM，因此代码能安全地延迟执行。

```html
<script type="text/javascript" src="demo_defer.js" defer="defer"></script>
```

JavaScript将在页面执行到带有defer属性的&lt;script>标签时开始下载，但不会马上执行，直到DOM加载完成之后(onload事件被触发前)，因此它不会阻塞浏览器的其他进程，这类文件可以与页面中的其他资源并行下载。

补充说明：

**HTML5 defer 属性**

defer 属性仅适用于外部脚本（只有在使用 src 属性时）。

有多种执行外部脚本的方法：

- 如果 async="async"：脚本相对于页面的其余部分异步地执行（当页面继续进行解析时，脚本将被执行）
- 如果不使用 async 且 defer="defer"：脚本将在页面完成解析时执行
- 如果既不使用 async 也不使用 defer：在浏览器继续解析页面之前，立即读取并执行脚本


### 2.动态脚本元素

可以用JavaScript动态创建HTML汇总几乎所有内容，包括&lt;script>标签。

```javascript
var script = document.createElement('script');
script.type = 'text/javascript';
script.src = 'file1.js';
document.getElementByTagName('head')[0].appendChild(script);
```

这种技术的重点在于：无论何时启动下载，文件的下载和执行过程不会阻塞页面其他进程，可以将代码放在页面<head>区域而不影响页面其他部分。

使用动态脚本节点下载文件后，返回的代码通常会立即执行，此时必须确保脚本下载完成并准备就绪，通过动态&lt;script>节点触发的事件来实现。

Firefox、Opera、Chrome和Safari3以上版本会在&lt;scirpt>元素接收完成时触发一个load事件，可通过侦听此事件获得脚本加载完成状态。IE的实现方式是它会触发一个readystatechange事件。&lt;script>元素提供一个readyState属性(有'uninitialized','loading','loaded','interactive','comlete'5种取值)，当'loaded'或'complete'触发时执行回调并删除事件处理器（以确保事件不会处理两次）。


下面一个是动态加载JavaScript文件的函数（封装了标准及IE特有的实现方式）,使用了特征检测(我觉得这个...更像是浏览器推断呀！？)：

```javascript
function loadScript(url,callback) {
	
	var script = document.createElement('script');
	script.type = 'text/javascript';
	
	if (script.readyState) { // IE
		script.onreadystatechange = function() {
			if (script.readyState === 'loaded' || script.readyState === 'complete') {
				script.onreadystatechange = null;
				callback();
			}
		};
	} else { //其他浏览器
		script.onload = function() {
			callback();
		};
	}

	script.src = url;
	document.getElementByTagName('head')[0].appendChild(script);
}

// 使用
loadScript('file1', function() {
	alert('File is loaded');
});
```


### 3.XMLHttpRequest脚本注入

使用XMLHTTPRequest(XHR)对象获取脚本并注入页面。此技术先创建一个XHR对象，然后用它下载JavaScript文件，最后通过动态&lt;script>元素将代码注入页面。

```javascript
var xhr = new XMLHttpRequest();
xhr.open('get', 'file1.js', true);
xhr.onreadystatechange = function() {
	if (xhr.readyState === 4) {
		if (xhr.status >= 200 && xhr.status < 300 || xhr.status === 304) {
			var script = document.createElement('script');
			script.type = 'text/javascript';
			script.text = xhr.responseText;
			document.body.appendChild(script);
		}
	}
};
xhr.send(null);
```
　
优点：可以下载JavaScript代码但不是立即执行，主流浏览器都支持

局限性：JavaScript文件必须与所请求的页面处于相同的域，因此大型的Web应用通常不会采用XHR脚本注入技术。


