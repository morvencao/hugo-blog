---
title: "Javascript模块化开发"
date: 2016-10-16
type: "notes"
draft: false
---

随着互联网时代的到来，前端技术更新速度越来越快。起初只要在script标签中嵌入几行代码就能实现一些基本的用户交互，到现在随着Ajax，jQuery，MVC以及MVVM的发展，Javascript代码量变得日益庞大复杂。
网页越来越像桌面程序，需要一个团队分工协作、进度管理、单元测试等等......开发者不得不使用软件工程的方法，管理网页的业务逻辑。
Javascript模块化开发，已经成为一个迫切的需求。理想情况下，开发者只需要实现核心的业务逻辑，其他都可以加载别人已经写好的模块。
但是，Javascript不是一种模块化编程语言，它不支持"类"（class），更遑论"模块"（module）了。直到前不久ES6正式定稿，Javascript才开始正式支持"类"和"模块"，但还需要很长时间才能完全投入实用。

### 什么是模块化

模块是任何大型应用程序架构中不可缺少的一部分，一个模块就是实现特定功能的代码区块或者文件。模块可以使我们清晰地分离和组织项目中的代码单元。在项目开发中，通过移除依赖，松耦合可以使应用程序的可维护性更强。有了模块，开发者就可以更方便地使用别人的代码，想要什么功能，就加载什么模块。模块开发需要遵循一定的规范，否则就会混乱不堪。

Javascript社区做了很多努力，在现有的运行环境中，实现"模块"的效果。本文总结了当前＂Javascript模块化编程＂的最佳实践，说明如何投入实用。

### Javascript模块化基本写法

在第一部分，将讨论基于传统Javascript语法的模块化写法。

#### 原始写法

模块就是实现特定功能的一组方法。
只要把不同的函数（以及记录状态的变量）简单地放在一起，就算是一个模块。
```javascript
function func1(){
	//...
}

function func2(){
	//...
}
```

上面的函数func1()和func2()，组成一个模块。使用的时候，直接调用就行了。
这种做法的缺点很明显："污染"了全局变量，无法保证不与其他模块发生变量名冲突，而且模块成员之间看不出直接关系。

#### 对象写法

为了解决上面的缺点，可以把模块写成一个对象，所有的模块成员都放到这个对象里面。
```javascript
var moduleA = new Object({
	_count : 0,
	func1 : function (){
		//...
	},
	func2 : function (){
		//...
	}
});
```

上面的函数func1()和func2(），都封装在moduleA对象里。使用的时候，就是调用这个对象的属性。
```javascript
moduleA.func1();
```

但是，这样的写法会暴露所有模块成员，内部状态可以被外部改写。比如，外部代码可以直接改变内部计数器的值。
```javascript
moduleA._count = 3;
```

#### 立即执行函数写法

使用"立即执行函数"（Immediately-Invoked Function Expression，IIFE），可以达到不暴露私有成员的目的。
```javascript
var moduleA =  (function(){
	var _count = 0;
	var func1 = function(){
		//...
	};
	var func2 = function(){
		//...
	};
	return {
		func1 : func1,
		func2 : func2
	};
})();
```

使用上面的写法，外部代码无法读取内部的_count变量。

```javascript
console.log(moduleA._count);    //undefined
```

moduleA就是Javascript模块的基本写法。接下来，对这种写法进行再加工。

#### 放大模式

如果一个模块很大，必须分成几个部分，或者一个模块需要继承另一个模块，这时就有必要采用"放大模式"（augmentation）。
```javascript
var moduleA = (function (mod){
	mod.func3 = function () {
		//...
	};
	return mod;
})(moduleA);
```

上面的代码为moduleA模块添加了一个新方法func3()，然后返回新的moduleA模块。

#### 宽放大模式（Loose augmentation）

在浏览器环境中，模块的各个部分通常都是从网上获取的，有时无法知道哪个部分会先加载。如果采用上一节的写法，第一个执行的部分有可能加载一个不存在空对象，这时就要采用"宽放大模式"。
```javascript
var moduleA = ( function (mod){
	//...
	return mod;
})(window.moduleA || {});
```

与"放大模式"相比，＂宽放大模式＂就是"立即执行函数"的参数可以是空对象。

#### 输入全局变量

独立性是模块的重要特点，模块内部最好不与程序的其他部分直接交互。
为了在模块内部调用全局变量，必须显式地将其他变量输入模块。

```javascript
var moduleA = (function ($, YAHOO) {
	//...
})(jQuery, YAHOO);
```

上面的moduleA模块需要使用jQuery库和YUI库，就把这两个库（其实是两个模块）当作参数输入moduleA。这样做除了保证模块的独立性，还使得模块之间的依赖关系变得明显。这方面更多的讨论，参见Ben Cherry的著名文章[《JavaScript Module Pattern: In-Depth》](http://www.adequatelygood.com/2010/3/JavaScript-Module-Pattern-In-Depth)。

### 基于规范的Javascript模块化

为了让开发者都以同样的方式编写模块，必须制定模块的规范。虽然到目前为止Javascript模块还没有官方规范，但通行的Javascript模块规范共有两种：CommonJS和AMD。

#### CommonJS

2009年，美国程序员Ryan Dahl创造了[node.js](http://nodejs.org/)项目，将javascript语言用于服务器端编程。
这标志"Javascript模块化编程"正式诞生。因为老实说，在浏览器环境下，没有模块也不是特别大的问题，毕竟网页程序的复杂性有限；但是在服务器端，一定要有模块，与操作系统和其他应用程序互动，否则根本没法编程。

node.js的[模块系统](http://nodejs.org/docs/latest/api/modules.html)，就是参照[CommonJS规范](http://wiki.commonjs.org/wiki/Modules/1.1)实现的。
根据CommonJS规范，一个单独的文件就是一个模块。每一个模块都是一个单独的作用域，也就是说，在该模块内部定义的变量，无法被其他模块读取，除非定义为global对象的属性。输出模块变量的最好方法是使用module.exports对象。
```javascript
var i = 1;
var max = 30;

module.exports = function () {
	for (i -= 1; i++ < max; ) {
		console.log(i);
	}
	max *= 1.1;
};
```

上面代码通过module.exports对象，定义了一个函数，该函数就是模块外部与内部通信的桥梁。加载模块使用require方法，该方法读取一个文件并执行，最后返回文件内部的module.exports对象。

有了服务器端模块以后，很自然地，大家就想要客户端模块。而且最好两者能够兼容，一个模块不用修改，在服务器和浏览器都可以运行。
但是，问题来了。
```javascript
var math = require('math');
math.add(2, 3);
```
第二行math.add(2, 3)，在第一行require('math')之后运行，因此必须等math.js加载完成。也就是说，如果加载时间很长，整个应用就会停在那里等。
这对服务器端不是一个问题，因为所有的模块都存放在本地硬盘，可以同步加载完成，等待时间就是硬盘的读取时间。但是，对于浏览器，这却是一个大问题，因为模块都放在服务器端，等待时间取决于网速的快慢，可能要等很长时间，浏览器处于"假死"状态。
因此，浏览器端的模块，不能采用"同步加载"（synchronous），只能采用"异步加载"（asynchronous）。这就是AMD规范诞生的背景。

#### AMD

[AMD](https://github.com/amdjs/amdjs-api/wiki/AMD)是"Asynchronous Module Definition"的缩写，意思就是"异步模块定义"。它采用异步方式加载模块，模块的加载不影响它后面语句的运行。所有依赖这个模块的语句，都定义在一个回调函数中，等到加载完成之后，这个回调函数才会运行。

AMD也采用require()语句加载模块，但是不同于CommonJS，它要求两个参数：
```javascript
require([module], callback);
```

第一个参数[module]，是一个数组，里面的成员就是要加载的模块；第二个参数callback，则是加载成功之后的回调函数。如果将前面的代码改写成AMD形式，就是下面这样：

```javascript
require(['math'], function (math) {
	math.add(2, 3);
});
```

math.add()与math模块加载不是同步的，浏览器不会发生假死。所以很显然，AMD比较适合浏览器环境。
目前，主要有两个Javascript库实现了AMD规范：[require.js](http://requirejs.org/)和[curl.js](https://github.com/cujojs/curl)。

#### requireJS

AMD是一个在浏览器端模块化开发的规范，由于不是JavaScript原生支持，使用AMD规范进行页面开发需要用到对应的库函数，也就是大名鼎鼎RequireJS，实际上AMD 是 RequireJS 在推广过程中对模块定义的规范化的 产出。
requireJS的诞生，就是为了解决两个问题：
1. 多个js文件可能有依赖关系，被依赖的文件需要早于依赖它的文件加载到浏览器
2. js加载的时候浏览器会停止页面渲染，加载文件越多，页面失去响应时间越长

首先来看个例子：
定义模块：（moduleA.js）
```javascript
// define moduleA.js
define(['myLib'], function(myLib) {
	function foo() {
		myLib.doSomething();
	}
	return {
		foo : foo
	};
});
```

加载模块并调用：
```javascript
// load moduleA
require(['moduleA'], function (m) {
	m.foo();
}
```

requireJS的语法
requireJS定义了一个函数 define，它是全局变量，用来定义模块：
```javascript
define(id?, dependencies?, factory);
```
id：可选参数，用来定义模块的标识，如果没有提供该参数，脚本文件名（去掉拓展名）
dependencies：是一个当前模块依赖的模块名称数组
factory：工厂方法，模块初始化要执行的函数或对象。如果为函数，它应该只被执行一次。如果是对象，此对象应该为模块的输出值

在页面上使用`require`函数加载模块
```javascript
require([dependencies], function(){});
```

require()函数接受两个参数：
第一个参数是一个数组，表示所依赖的模块
第二个参数是一个回调函数，当前面指定的模块都加载成功后，它将被调用。加载的模块会以参数形式传入该函数，从而在回调函数内部就可以使用这些模块
require()函数在加载依赖的函数的时候是异步加载的，这样浏览器不会失去响应。同时它指定的回调函数，只有前面的模块都加载成功后，才会运行，解决了依赖性的问题。

#### CMD

CMD(Common Module Definition)，即通用模块定义，该规范明确了模块的基本书写格式和基本交互规则。该规范是在国内发展出来的。AMD是依赖关系前置，CMD是按需加载。
[CMD规范](https://github.com/seajs/seajs/issues/242)是国内发展出来的，就像AMD有个requireJS，CMD有个浏览器的实现SeaJS，SeaJS要解决的问题和requireJS一样，只不过在模块定义方式和模块加载（可以说运行、解析）时机上有所不同。

语法
Sea.js 推崇一个模块一个文件，遵循统一的写法

模块定义：
```javascript
define(id?, deps?, factory);
```

因为CMD规范推崇一个文件即一个模块，所以经常就用文件名作为模块id。CMD主张依赖就近，所以一般不在`define`的参数中写依赖。`factory`表示是模块的构造方法。执行该构造方法，可以得到模块向外提供的接口。`factory`方法在执行时，默认会传入三个参数：require、exports 和 module
```javascript
define(function(require, exports, module) {
	// module code...
});
```

require：
require 是factory函数的第一个参数，是一个方法，接受模块标识作为唯一参数，用来获取其他模块提供的接口。
```javascript
require(id)
```
exports：
exports 是一个对象，用来向外提供模块接口。

module
module 是一个对象，上面存储了与当前模块相关联的一些属性和方法。

下面是一个符合CMD规范的一个例子：
```javascript
// define moduleA.js
define(function(require, exports, module) {
	var $ = require('jquery.js');
	$('div').addClass('active');
});
```

```javascript
// load module
seajs.use(['moduleA.js'], function(m) {
	// ...
});
```

#### AMD与CMD区别

两种规范的制定初衷不同：
> AMD 是 RequireJS 在推广过程中对模块定义的规范化产出。
> CMD 是 SeaJS 在推广过程中对模块定义的规范化产出。

对于依赖的模块，AMD 是提前加载，CMD 是延迟加载：
> AMD推崇依赖前置，在定义模块的时候就要声明其依赖的模块
> CMD推崇就近依赖，只有在用到某个模块的时候再去加载

这种区别各有优劣，只是语法上的差距，而且requireJS和SeaJS都支持对方的写法。

AMD和CMD最大的区别是对依赖模块的执行时机处理不同，注意不是加载的时机或者方式不同

很多人说requireJS是异步加载模块，SeaJS是同步加载模块，这种理解实际上是不准确的。其实两种规范加载依赖模块都是异步的，只不过AMD依赖前置，factory可以方便知道依赖模块有哪些，而CMD就近依赖，需要使用时把模块变为字符串解析一遍才知道依赖了哪些模块，这也是很多人诟病CMD的一点，牺牲性能来带来开发的便利性，实际上解析模块用的时间短到可以忽略。

同样都是异步加载模块，AMD在加载模块依赖后就会执行该模块，所有模块都加载执行完后会进入require的回调函数，执行主逻辑。这样的效果就是依赖模块的执行顺序和书写顺序不一定一致，取决于加载模块的速度，哪个先加载，哪个就先执行，但是主逻辑一定在所有依赖加载完成后才执行。

CMD加载完某个依赖模块后并不执行，只是下载而已，在所有依赖模块加载完成后进入主逻辑，遇到require语句的时候才执行对应的模块，这样模块的执行顺序和书写顺序是完全一致的。

这也是很多人说AMD用户体验好，因为没有延迟，依赖模块提前执行了，CMD性能好，因为只有用户需要的时候才执行的原因。

#### 面向未来的ES6模块化标准

既然模块化开发的呼声这么高，作为官方的ECMA必然要有所行动。其实ECMA很早就把模块化列入草案，终于在2015年6月份发布的ES6正式版中包含了模块化的标准规范。然而，可能由于所涉及的技术还未成熟，ES6移除了关于模块如何加载/执行的内容，只保留了定义、引入模块的语法。所以说现在的ES6 Module还只是个雏形，半成品都算不上。但是这并不妨碍我们先窥探一下ES6模块标准。
定义一个模块不需要专门的工作，因为一个模块的作用就是对外提供API，ES6规定可以用module关键字定义一个模块，对于模块需要对外提供的API只需用exoprt导出就可以：

```javascript
// Method 1: 
export var a = 1;
export var obj = {name: 'abc', age: 20};
export function run() {....}
```
```javascript
// Method 2:
var a = 1;
var obj = {name: 'abc', age: 20};
function run() {....}
export {a, obj, run}
```
```javascript
// Method 3:
module math {
	export function sum(x, y) {
		return x + y;
	}
	export var pi = 3.141593;
}
```

使用import关键字来加载外部模块：

```javascript
// we can import in script code, not just inside a module
import {sum, pi} from math;
alert("2π = " + sum(pi, pi));
```

```javascript
// import everything
import * from math;
alert("2π = " + sum(pi, pi));
```

```javascript
// import part of module
import {run as go} from  'a';
import { draw: drawShape } from shape;
run();
drawShape();
```

```javascript
// nested module
module widgets {
	export module button { ... }
	export module alert { ... }
	export module textarea { ... }
	...
}
```
 
从服务器上请求的模块：
```javascript
<script type=”harmony”>
// loading from a URL
module JSON at 'http://json.org/modules/json2.js';
alert(JSON.stringify({'hi': ‘world'}));
```

动态载入一个模块：
```javascript
Loader.load('http://json.org/modules/json2.js', function(JSON) {
	alert(JSON.stringify([0, {a: true}]));
});
```

ES6 Module的基本用法就是这样，可以看到确实是有些薄弱，还需要很长时间来规范化，可谓任重而道远。且它有个问题，即新的语法关键字不能向下兼容（如低版本IE浏览器）。目前我们可以使用一些第三方模块来对ES6进行编译，转化为可以使用的ES5代码，或者是符合AMD规范的模块，例如ES6 module transpiler。另外有一个项目也提供了加载ES6模块的方法，比如[es6-module-loader](https://github.com/ModuleLoader/es6-module-loader)，不过这都是一些临时的方案，或许明年ES7一发布，模块的加载有了标准，浏览器给与了实现，这些工具也就没有用武之地了。

未来还是很值得期待的，从语言的标准上支持模块化，js就可以更加自信的走进大规模企业级开发。