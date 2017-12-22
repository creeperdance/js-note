# Ajax

Ajax可以通过延迟下载体积较大的资源文件来使得页面加载速度更快。它通过异步的方式在客户端和服务器间传输数据，使网页实现异步更新，即它可以让你在不重新加载整个页面的情况下完成对部分页面的更新。

选择合适的传输方式和最有效的数据格式，可以显著改善用户和网站的交互体验。


## 数据传输

Ajax可以和服务器通信而无需重新加载页面，数据可以从服务器获取或发送给服务器。本节会讨论建立这种通信通道的多种方法和其性能。


### 请求数据

**5种常用于向服务器请求数据的技术：**

- XMLHttpRequest(XHR)
- Dynamic script tag insertion 动态脚本注入
- iframes
- Comet
- MultipartXHR



#### XMLHttpRequest

**GET请求：**


```javascript
var url = '/data.php';
var params = [
    'id=934875',
    'limit=20'
];

var req = new XMLHttpRequest();

req.onreadystatechange = function(){

    // 正在`流`数据 接受到部分信息,但不是所有
    if (req.readyState === 3) {
        var data_now = req.responseText; // 获取数据
       	
    }

    if (req.readyState === 4) {
        var reponse_headers = req.getAllReponseHeaders(); // 获取响应头信息

        var data = req.reponseText; // 获取数据
        // 数据处理
    }

    req.open('GET',url+'?' +params.join('&'), true);
    req.setRequestHeader('X-Requested-With', ' XMLHttpRequest'); // 设置请求头信息
    req.send(null); // 发送请求,GET方式通过send提交数据,服务器接收不到

}
```


**POST方式：**

```javascript

function xhrPost(url, params, callback){
    var req = XMLHttpRequest();

    req.onreadystatechange = function() {
        if (req.readyState ==4) {
            if (callback && typeof callback === 'function') {
                callback();
            }
        }
    };

    req.open('POST', url, true);
    req.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    req.setRequestHeader('Contetn-Length', params.length);
    req.send(params.join('&'));
}

```

【注意：】

- 不能使用XHR从外域请求数据
- GET不改变服务器状态，只会获取数据，经GET请求的数据会被缓存起来，多次请求同一数据建议用它。当URL长度超过2048时,建议使用POST。

#### 动态脚本注入

优点：能跨域请求数据，速度快。

缺点：

- 不能设置请求头。
- 只能用GET方式请求。
- 不能设置请求超时处理或重试，就算请求失败了也不一定知道。
- 等数据都返回才可以访问。
- 不能访问请求头信息，不能将响应消息作为字符串处理。
- 响应消息作为脚本标签源码必须为可执行JavaScript代码，数据最好封装在回调函数中使用。
- 使用外域的JSON要格外小心，用此方式添加到页面的任何代码都可完全控制整个页面，容易有安全问题。


```javascript
var scriptElement = document.createElement('script');
scriptElement.src = 'http://any-domain.com/javascript/lib.js';
document.getElementByTagName('head')[0].appenChild(scriptElement);

//不能使用纯XML，JSON或其他任何格式的数据，需要封装在回调函数中
function jsonCallback(jsonString) {
	var data = eval('(' + jsonString + ')');
	// 处理数据...
}

```

外部lib.js:

```javascript
// 当该文件加载完成后，调用jsonCallback()方法
jsonCallback( {'status': 1, 'colors': ['#fff', '#000', '#ff0000'] });
```

#### MultipartXHR (MXHR)

multipartXHR允许客户端只用一个HTTP请求就可以从服务器向客户端传送多个资源。

此技术比各自独立请求的方式快4~10倍。

其实就是服务器将资源（CSS文件、HTML片段、JavaScript代码或base64编码的图片）打包成一个双方约定的字符串分割的长字符串再发送给客户端。让JavaScript处理这个长字符串，并根据mine-type类型和传入的其他“头信息”解析每个资源。

下面的函数用于将JavaScript代码、CSS样式和图片转换为浏览器可用资源：

```javascript
function handleImageData(data,mineType) {
	var img = document.createElement('img');
	img.src = 'data:' + mineType + ';base64,' + data;
	return img;
}

function handleCss(data) {
	var style = document.createElement('style');
	style.type = 'text/css';
	
	var node = document.createTextNode(data);
	style.appendChild(node);
	document.getElementByTagName('head')[0].appendChild(style);
}

function handleJavaScript(data) {
	eval(data);
}
```


由于MXHR响应消息体积越来越大，因此有必要在每个资源收到的时候就立即处理，而不是等整个响应消息接收完成。**可以通过监听readyState为3的状态实现。**


```javascript
var req = new XMLHttpRequest(),
	getLatestPacketInterval, 
	lastLength = 0;

req.open('GET', 'rollup_images.php', true);
req.onreadystatechange = readyStateHandler;
req.send(null);

function readyStateHandler () {
	if (req.readyState === 3 && getLatestPacketInterval === null) {
		// 开始轮询
		getLatestPacketInterval = window.setInterval(function() {
			getLatestPacket();
		}, 15);
	}

	if (req.readyState === 4) {
		// 停止轮询
		clearInterval(getLatestPacketInterval);
		
		// 获取最后一个数据包
		getLastestPacket();
	}	
}

function getLatestPacket() {
	var length = req.responseText.length;
	var packet = req.responseText.substring(lastLength,length);

	processPacket(packet);
	lastLength = length;
}
```



健壮的MXHR代码见：<a href="http://techfoolery.com/mxhr/">http://techfoolery.com/mxhr/</a>。

【注意：】

- 以这种方式获取的资源不能被浏览器缓存
- 页面包含了大量其他地方用不到的资源。
- 无需从缓存读数据，除非重载页面。

### 发送数据

两种广泛使用的将数据发送给服务器的技术：

- XMLHTTPRequest
- 信标(beacons)



#### XMLHttpRequest

上面已经介绍过XMLHttpRequest,在上述代码中对于发送失败的情况并没有做处理，因此可以添加如下代码：

```javascript
req.onerror = function() {
	setTimeout(function() {
		xhrPost(url, params, callback);
	}, 1000);
};
```

【注意：】

**GET请求服务器只发送一个TCP数据包，POST将发送两个TCP数据包。**对于GET，浏览器将header和data一并发送，服务器响应200；而对于POST,浏览器会先发送header,服务器响应100 continue,再发送data，服务器响应200。



#### Beacons

Beacons非常类似于动态脚本注入。

```
var url = '/status_tracker.php';
var params = [
    'step=2',
    'time=1248219291',
];


(new Image()).src = url + '?' + params.join('&');

// 需要获取返回信息
var beacon  = new Image();
beacon.src = url + '?' + params.join('&');

// 假设服务器会返回图片,设定返回图片宽度为1表达“成功”,2为“失败”
beacon.onload = function (){
    if(this.width === 1){
        // 成功
    }else if (this.width === 2 ){
        // 失败,重试并创建另一个信标
    }
};

beacon.onerror = function() {
	// 出错，稍后重试并创建另一个信标
}

```

【注意：】

- 如果不需要在响应中返回数据,服务器应该返回一个204状态码,阻止客户端继续等待不会到来的消息正文.
- 这个是给服务器回传消息的最佳方式,性能消耗很小
- 服务器的错误完全影响不到客户端
- 唯一缺陷，和GET一样受url长度限制


## 数据格式

### XML

优点：格式严格，易于验证

缺点：冗长，每个数据片依赖大量结构，解析繁琐。

如果只有XML格式的数据可用再考虑它。

### JSON

- JSON是一种使用JavaScript对象和数组直接量编写的轻量级且易于解析的数据格式。

- JSON数据就是一段可执行的JavaScript代码(尽可能用JSON.parse()方法解析而不要用eval)。

- JSON-P数据使用动态脚本注入获取，它将数据当做可执行的JavaScript而不是字符串，解析数据极快。它能跨域使用，但涉及敏感数据时不要用它。


### HTML

一种将在服务器上搭建好的整个HMTL传回客户端的技术，JavaScript可以直接通过innerHTML将它插入到页面响应位置。

带来的问题：

HTML是一种臃肿的数据格式，比XML更繁杂，解析时间长。

### 自定义格式

可以自定义一种只简单把数据用分隔符连接起来的格式，通过这些分隔符可以将其转换为数组，不同的分隔符可以创建多维数组。

分隔符最好采用不存在于数据中的一个单字符。

**使用XHR和动态脚本注入获取，用split()解析，解析大数据集时比JSON-P快，通常文件尺寸小。**

## Ajax性能指南

### 缓存数据

最快的Ajax请求就是没有请求，有两种方法可以避免发送不必要的请求：

- 在服务端，设置HTTP头部信息以确保你的响应会被浏览器缓存。简单好维护。
- 在客户端，把获取到的信息存储到本地，从而避免再次请求。好控制。

**设置HTTP头部信息**

若希望Ajax响应能被浏览器缓存，必须使用GET方式发请求。而且必须在响应中发送正确的HTTP头信息。Expires头信息会告诉浏览器应该缓存响应多久，其值为一个日期，过期后对该URL的任何请求都不再从缓存中获取，而是重新访问服务器。

Expires头信息格式(缓存到2014年7月)：

	Expires: Mon, 28 Jul 2014 23:30:00 GMT


确保浏览器缓存最简单的方法，无需改变客户端代码,缓存内容可以跨页面跨回话(session)。



**本地数据存储**

手工缓存。直接把服务器接收到的数据储存起来。将响应文本保存到一个对象中，以URL为键值作为索引。

手动缓存可以更好地工作在移动设备上，此类设备大多数浏览器的缓存都很小或没有缓存。

### 了解Ajax类库的局限

兼容各版本获取XHR对象：

```javascript
function createXhrObject() {
	var msxml_progid = [
		'MSXML2.XMLHTTP.6.0',
		'MSXML3.XMLHTTP',
		'Microsoft.XMLHTTP', // 不支持readyState3
		'MSXML2.XMLHTTP.3.0', // 不支持readyState3
	];
	
	var req;
	try {
		req = new XMLHttpRequest(); // 先尝试标准方法
	} catch(e) {
		for (var i = 0, len = msxml_progid.length; i < len; i++) {
			try {
				req = new ActiveXObject(msxml_progid[i]);
				break;
			} catch(e2) {
				
			}
		}
	} finally {
		return req;
	}
}
```