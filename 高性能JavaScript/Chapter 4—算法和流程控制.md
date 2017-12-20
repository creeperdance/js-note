# 算法和流程控制

## 循环

### 循环性能


#### 1. 减少迭代的工作量


```javascript
// 不好的写法
for (var i = 0;i < items.length;i++) {
	process(items[i]);
}
```

在上述循环中，每次运行循环体都会产生如下操作：

1. 在控制条件中查找一次属性（items.length)。
2. 在控制条件中执行一次数值比较（i < items.length)。
3. 一次比较操作查看控制条件的计算结果是否为true（i < items.length === true)。
4. 一次自增操作(i++)。
5. 一次数组查找(items[i])。
6. 一次函数调用（ process(items[i]) ）。

这个例子中，每次循环都要查找items.length，这样做很耗时，该值在循环运行过程中并没改变，因此产生了不必要的性能损耗。因此，只查找一次属性，并把值存储到一个局部变量，然后在控制条件中使用这个变量。

```javascript
// 最小化属性查找
for (var i = 0, len = items.length; i < len; i++) {
	process(items[i]);
}
```


#### 2. 减少迭代次数

**达夫设备(Duff's Device)**

一个典型实现：

```javascript
// 书中代码有误：Math.floor应当改为Math.ceil
var iterations = Math.floor(items.length / 8),
	startAt = items.length / 8,
	i = 0;

do {
	switch (startAt) {
		case 0: process(items[i++])；
		case 7: process(items[i++])；
		case 6: process(items[i++])；
		case 5: process(items[i++])；
		case 4: process(items[i++])；
		case 3: process(items[i++])；
		case 2: process(items[i++])；
		case 1: process(items[i++])；
	}
	startAt = 0;
} while(iterations);
```

达夫设备背后的理念是：每次循环中最多可以调用8次process()。循环的迭代次数总是除以8，变量startAt用来放余数，表示第一次循环应调用多少次process()。如果是12次，第一次调用process()4次，第二次调用process()8次，用两次循环替代了12次循环。

【注意，经过测试发现，书中代码iterations = Math.floor(items.length / 8)有误，应该改用Math.ceil()方法，否则只进行第一次调用process()4次，未进入第二次调用proces()8次。】


此算法有一个取消了switch的稍块版本：

```javascript
// 经过测试发现，下面代码有误，是死循环
var i = items.length % 8;
while (i) {
	process(items[i--]);
}

i = Math.floor(items.length / 8);

while (i) {
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	//i经过自减可能变成负数，导致while始终未true，无法跳出循环
}
```

【注意：书中代码有误，我对它做了一些修改，代码如下：】

```javascript
// 修改后的版本
var i = 0,
	startAt = items.length % 8,
	iterations = Math.floor(items.length / 8);

while (startAt--) {
	process(items[i++]);
}

while (iterations--) {
	process(items[i++]);
	process(items[i++]);
	process(items[i++]);
	process(items[i++]);
	process(items[i++]);
	process(items[i++]);
	process(items[i++]);
	process(items[i++]);
}
```

【必要性：

- 在迭代次数少的情况下，完全没必要用达夫设备进行优化
- 在老版本的浏览器中用达夫设备优化性能确实能大幅度提升，但新版浏览器对循环迭代语句进行了更强的优化，达夫设备能实现的优化效果也日渐减少。

】

### 基于函数的迭代


**forEach**

```javascript
items.forEach(funcion(value, index, array) {
	process(value);
});
```

尽管基于函数的迭代提供了更便利的迭代方法，但它仍然比基于循环的迭代要慢，对每个数组调用外部方法所带来的开销是速度慢的主要原因。

## 条件语句

### if-else对比switch

在条件增加时，if-else的性能负担增加的程度比switch要多。
**因此，更倾向于在条件数量较少时使用if-else，在条件数量较大时使用switch。**


### 优化if-else

优化目标：**最小化到达正确分支前所需判断的条件数量。**

- if-else语句的条件语句应该总是按照最大概率到最小概率的排列，以确保运行速度最快。
- 为了最小化条件判断的次数，经常需要使用一系列嵌套语句，划分区间，缩小范围。



### 查找表

有时候优化条件语句的最佳方案就是避免使用if-else和switch。当有大量离散值需要测试时，使用if-else和switch比使用查找表慢得多。

JavaScript中可以使用数组和普通对象来构建查找表，通过查找表访问数据比if-else和switch快很多，特别在条件语句数量很多的时候。

```javascript
// 作者的代码中并无break，但我觉得这里的语义是需要break的。
switch(value) {
	case 0: return result0;break;
	case 1: return result1;break;
	case 2: return result2;break;
	case 3: return result3;break;
	case 4: return result4;break;
	case 5: return result5;break;
	case 6: return result6;break;
	case 7: return result7;break;
	case 8: return result8;break;
	case 9: return result9;break;
	case default: return result10;
}
```

上述的整个结构可以用一个数组的查找表替代：

```javascript
// 将返回值集合存入数组
var results = [result0,result1,result2,result3,result4,result5,
			   result6,result7,result8,result9,result10];

// 返回当前结果
return result[value];
```

查找表的优点：

- 不用书写任何条件判断语句，即便候选值数量增多，也几乎不会产生额外的性能开销。
- 当单个键和单个值之间存在逻辑映射时，查找表达优势就体现出来了。switch更适用于每个键都需要对应一个独特的动作或一系列动作的场合。




## 递归

使用递归可以把复杂的算法变简单。比如阶乘函数：

```javascript
function factorial(n) {
	if (n === 0) {
		return 1;
	} else {
		return n*factorial(n-1);
	}
}
```

递归函数的潜在问题是：终止条件不明确或缺失时，会导致函数长时间运行，并使得用户界面处于假死状态。而且，递归函数还可能遇到浏览器的“调用栈大小限制”(call stack size limit)。

### 调用栈限制

JavaScript引擎支持的递归数量与JavaScript调用栈大小直接相关。 除IE的调用栈是与系统空闲内存有关，其他浏览器都有固定数量的调用栈限制。

### 递归模式

栈溢出错误会导致其他代码中断运行，当你遇到栈溢出错误时，可将方法改写为迭代算法，或使用Memoization来避免重复计算。



### 迭代
 
任何递归能实现的算法同样可以用迭代来实现。

### Memoization

Memoization是避免重复工作的一种方法，它缓存前一个计算结果供后续计算使用，避免了重复工作。


（详细见《JavaScript语言精粹》——函数篇）