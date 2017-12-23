# 编程实践

## 避免双重求值

标准动态执行代码方法：

```javascript
var num1 = 5,
	num2 = 6;
```

**1. eval()**

```javascript
result = eval('num1 + num2');
```

**2. Function()构造函数**

```javascript
sum = new Function('arg1', 'arg2', 'return arg1 + arg2');
```

**3. setTimeout()**

```javascript
setTimeout('sum = num1 + num2', 100);
```

**4. setInterval()**

```javascript
setInterval('sum = num1 + num2', 100);
```

在执行一段JavaScript代码时去执行另外一个JavaScript，会导致双重求值的性能消耗，最好上面4种都不要用。


## 使用Object/Array直接量

使用对象和数组直接量是创建对象和数组最快的方式。

```javascript
// 一般方法
var myObject = new Object();
myObject.name ='Nicholas';
myObject.count = 50;


var myArray = new Array();
myArray[0] = 1;
myArray[1] = 2;


// 推荐方法：使用字面量方式
var myObject = {
    'name' : 'Nicholas'
};

var myArray = ['1',2];
```



## 避免重复工作

- 别做无关紧要的工作
- 别重复做已经完成的工作


最常见的重复工作：

```javascript
function addHandler(target, eventType, handler) {
	if (target.addEventListener) { // DOM2 Events
		target.addEventListener(eventType, handler, false);
	} else { // IE
		target.attachEvent('on' + eventType, handler);
	}
}

function removeHandler(target, eventType, handler) {
	if (target.removeEventListener) {// DOM2 Events 
		target.removeEventListener(eventType, handler, false);
	} else { // IE
		target.detachEvent('on' + eventType, handler);
	}	
}
```

应该在第一次调用addHandler()就确定addEventListener()是否存在了，后面每次调用时它也都存在，所以每一次调用函数都重复判断是否存在是一种浪费。


**几种解决方法:**

### 1.延迟加载

延迟加载意味着在信息被使用前不会做任何操作。

对上面例子而言，在函数调用前，没必要判断用哪种方法去绑定或取消绑定事件处理器。

```javascript
function addHandler(target, eventType, handler) {
	// 重复现有函数
	if (target.addEventListener) { // DOM2 Events
		addHandler = function (target, eventType, handler) {
			target.addEventListener(eventType, handler, false);
		};
	} else { // IE
		addHandler = function(targe, eventType, handler) {
			target.attachEvent('on' + eventType, handler);
		};
	}

	// 调用新函数
	addHandler(target, eventType, handler);
}
```

这个方法在第一次调用的时候会先判断，然后用新的函数覆盖，后面每次调用的时候就都调用新函数，不再执行判断逻辑（嗯嗯，很酷...）。

### 2.条件预加载

条件预加载会在脚本加载期间提前检测，而不会等到函数被调用后才检测。检测依然只进行一次。

```javascript
var addHandler = document.body.addEventListener ? 
				 function(target, eventType, handler) {
					target.addEventListener(eventType, handler, false);
				 } :
				 function(target, eventType, handler) {
					target.attachEvent('on' + eventType, handler);	
				 };
				 
```

## 位操作

JavaScript中的数字都依照IEEE-754标准以64位格式存储。在位操作中，数字被转换为有符号的32位格式，每次运算符胡直接操作该32位数以得到结果。

```javascript
var num = 25;
console.log(num.toString(2)); // 会忽略最高位的0
```


JavaScript有四种逻辑运算符：

- 按位与 &
- 按位或 |
- 按位异或 ^
- 按位取反 ~
- 左移 <<
- 右移 >>
- 无符号右移

```javascript
// &
console.log(25 & 3); //1

// |
console.log(25 | 3); //27

// ^
console.log(25 ^ 3); //26

// ~
console.log(25 ~ 3); //-26

```

几种利用位操作提升JavaScript速度的方法：

**1. 判断奇偶**

```javascript
// 对2取模运算实现表格行颜色交替
for(var i = 0, len = rows.length; i < len; i++) {
	if (i % 2) {
		className = 'even';
	} else {
		className = 'old';
	}
}


// 优化方法 
if(i & 2) { //偶数最低位都是0，奇数最低位都是1
	// ...
}

```


**2. 位掩码**

位掩码用于处理多个布尔选项的情况，用单个数字的每一位来判断选项中的每一项是否成立，掩码中每个项的值都等于2的幂。

```javascript
var OPTION_A = 1; 
var OPTION_B = 2; 
var OPTION_C = 4; 
var OPTION_D = 8; 
var OPTION_E = 16; 

// 定义可选项
var options =OPTION_A | OPTION_C | OPTION_D; 

// 选项A是否在列表中
if (options & OPTION_A) {
	
} else {

}

```

## 使用原生方法

无论代码如何优化都不会比原生方法快。JavaScript原生代码在写代码前就已经存在浏览器中了且都是用低级语言写的(如C++)。

