
# 快速响应的用户界面

JavaScript和用户界面更新在同一进程中运行，一次只能处理一件事情，所以JavaScript代码正在运行时，用户界面不能响应输入，反之亦然。因此，高效地管理UI进程就是要确保JavaScript不能运行太长时间，以免影响用户体验。

## 浏览器UI进程

用于执行JavaScript和更新用户界面的进程通常被称为“浏览器UI进程”。UI进程工作基于一个队列系统。在执行JavaScript代码时，大多数浏览器会停止把新的UI任务放入队列，比如你在JavaScript代码运行时点击按钮，浏览器可能不会把重绘按下的状态的任务或点击按钮启动的新JavaScript任务加入队列，最终失去一个响应。

浏览器限制了JavaScript的运行时间，以确保某些恶意代码不能通过永不停止的密集操作锁住用户的浏览器或计算机。此类限制有两种：**调用栈大小限制**和**长时间运行脚本限制**(浏览器将记录一个脚本的运行时间，在达到一定限制时终止它，并弹框提示之)。‘


**单个JavaScript操作花费的总时长不应超过100毫秒。**


## 使用定时器让出时间片段

对于不能再100毫秒或更短时间完成的JavaScript任务，最佳解决方案是让出UI线程的控制权，使得UI可以更新。即停止JavaScript代码，等UI线程更新后再继续执行JavaScript。**=>JavaScript定时器。**

### 定时器基础

**1. setTimeout(函数，毫秒)**

创建一个执行一次的定时器。

**2. setInterval(函数，毫秒)**

创建一个周期性重复运行的定时器。

```javascript
var button = document.getElementById('my-button');
button.onclick = function() {
	oneMethod();

	setTimeout(function() {
		// 这里写耗时操作
	},50);

	anotherMethod();
}
```

注意：

- 50ms从setTimeout()调用时开始计算，50ms后被添加到UI队列。
- 定时器的代码只有在创建它的函数(这里即onclick处理事件)执行完成后，才可能被执行。
- 如果twoMethod()执行了超过50ms,定时器代码能在函数执行完成后立即执行。
- 每创建一个定时器，都会让UI线程停止，并且会重置所有相关的浏览器限制。（包括长时间运行脚本定时器，调用栈也在定时器中重置为0）。

### 定时器精度

JavaScript定时器精度一般会差几ms,因此定时器不可用于测量实际时间。

延迟的最小值建议在：25ms,以确保至少有15ms延迟。


### 使用定时器操作数组

循环耗时过长会造成长时间运行脚本，可以将循环的工作分解到一系列定时器中以优化循环。

**是否可用定时器取代循环的决定性因素：**

- 处理过程是否必须同步？
- 数据是否必须按顺序处理？

如果这两个答案都是“否”，代码将适用于定时器分解任务。

一种基本的异步代码模式如下：

```javascript
function process_array(items,process,callback){
	// 克隆数组
    var tudo = items.concat();

    setTimeout(function(){
        process(todo.shift());

        if (todo.length > 0){
            // arguments.callee代表当前执行的匿名函数
            setTimeout(arguments.callee,25);
        } else {
            callback(items);
        }
    },25);
}
```

### 分割任务

如果一个函数运行时间太长，可以检查是否将它拆分为一系列能在较短时间内完成的子函数。

```javascript
function save_document(id){
    open_document(id);
    write_text(id);
    close_document(id);

    update_UI(id);
}


function multistep(steps,args,callback) {
	// 克隆数组
	var tasks = steps.concat();

	setTimeout (function() {
		// 执行下一个任务
		var task = tasks.shift();
	
		// 参数null或undefined，表示调用者为window
		task.apply(null,args || []);	
	
		if (tasks.length > 0) {
			setTimeout(arguments.callee, 25);
		} else {
			callback();
		}
	}, 25);

} 


// 使用
function save_document(id){
    var tasks = [open_doucment,write_text,close_document,update_UI];

    multistep(tasks,[id],function(){
        alert("Save completed!");
    });
}
```



### 定时器与性能

定时器能用来分割多个任务，使得JavaScript代码不用长时间阻塞UI队列。但多个重复的定时器同时创建往往会出现性能问题。因为只有一个UI进程，所有定时器都在争夺运行时间。

本节代码中使用的都是定时器序列，每执行完一个定时器才创建一个定义器，通过这种方式不会导致性能问题，不建议在一个函数定义多个同级定时器。

Thomas发现间隔1s以上的低频率重复定时器几乎不影响性能，而多个重复定时器使用较高频率（100ms~200ms)时，应用会变慢。

## Web Workers

Web Workers API 引入了一个接口，使代码运行且不占用浏览器UI线程的时间，它使得Web应用带来巨大性能提升，每个新的Worker都在自己的线程中运行，即Worker运行的代码不仅不会影响浏览器UI，也不会影响其它Worker中的代码。

### Worker 运行环境

Web Workers 没有绑定UI线程，所以它们不能访问浏览器中的许多资源，JavaScript和UI共享同一进程的部分原因是它们之间互相访问频繁。而Web Workers从外部线程中修改DOM会导致用户界面出错，每个Worker都有自己的全局运行环境，其功能只是JavaScript特性的一个子集，其运行环境由如下部分组成。

- 一个location对象(同window.location,不过所有属性都为只读）。
- self对象,指向全局的worker对象。
- 一个importScripts()方法,用来加载外部的javascript文件。
- 所有的ECMAScript对象，诸如:Object Array Date等。
- XMLHttpRequest构造器
- setTimeout方法 和 setInterval()方法
- 一个close()方法,能立马停止Worker.

Web Worker 有着不同的全局运行环境，因此无法从JavaScript代码中创建它，使用Worker需要创建一个完全独立的JavaScript文件，其中包含需要在Worker中运行的代码。要创建网页工人线程，必须传入这个JavaScript文件的URL：

```javascript
var worker = new Worker('code.js');
```

改代码一旦执行，将为该文件创建一个新的线程和一个新的Worker运行环境，该文件会被异步下载，直到文件下载并执行完成后才会启动此Worker。


### 与Worker通信

Worker与网页代码通过事件接口进行通信。

网页代码可以通过postMessage()方法给Worker传递数据，它接收一个参数，即需要传递给Worker的数据。

Worker有一个用来接收信息的onmessage事件处理器。

```javascript
// 外部js
var worker = new Worker('code.js');

// 用于接收Worker给外部JS传递的数据.
worker.onmessage = function(event) {
	alert(event.data);
};

//给Wroker发送数据
worker.postMessage('Nicholas');
```

Worker通过触发message事件接收数据，定义onmessage事件处理器后，改事件对象有一个data属性用于存放传入的数据。Worker可通过自己的postMessage()方法把信息传给页面。

```javascript
// code.js内部代码
self.onmessage = function(event) {
	self.postMessage('hello', + event.data + '!';
};

```

postMessage只支持原始值(字符串,数组,布尔,null,undefined),也可以传递Object和Array实例,有效数据会被序列化传入Worker,传出时反序列化.

### 加载外部文件

通过importScripts()方法加载外部JavaScript文件，将文件URL作为参数传递，该调用是阻塞式的，直到所有文件加载并执行完成后，脚本才会继续还运行，但不会也不能影响UI。


```javascript
importScripts(file_1.js,file_2.js);
```


Worker常见用法：

- 编码/解码大字符串
- 大数组排序.
- 复杂数学运算.


