# why

最初原因是作为面试准备的主题之一，后来发现研究原理最好的方法就是自己动手实现一个。

# what

前端早已进入了模块化开发的时代，Node.js火起来后，CommonJS作为服务端js模块化的标准。与此同时，针对浏览器端的模块化规范AMD, CMD陆续推出，其中最具代表性的就是require.js和sea.js。


# how

## 如何定位一个模块

每个module都有一个ID(一般是模块的相对路径),根据传入的baseURL+模块ID得到该模块的完整路径。

## 如何加载一个模块

得到模块的完整路径后，通过动态创建script脚本来加载模块：

```javascript
var node = document.createElement('script');
var head = document.getElementsByTagName('head')[0];

node.src = url;
node.async = true;

node.addEventListener('load', callback, false);

head.insertBefore(node, head.firstChild);
```

## 如何获取一个模块的依赖

每加载一个模块，都会在内部维护一个模块依赖映射表：
```javascript
{
	模块路径1：{
		id: '',
		dep: [], 依赖
		factory: fn  模块方法
	},
	模块路径2：{
		id: '',
		dep: [], 依赖
		factory: fn  模块方法
	}
	...
}
```
这样通过模块id很容易得到模块对应的依赖信息，找到依赖后，就可以通过id定位到模块的完整路径，然后通过完整路径去加载这个模块了，也就是重复加载单个模块的过程。


## 模块重复加载怎么办

开发过程中，很容易遇到A模块依赖B模块,C模块也依赖B模块的场景。如果按照上面的模块加载流程，B模块会被加载两次，也就是在分别加载A模块和C模块的时候，都会加载B。很明显这是一种资源上面的浪费，明明已经加载过了，为嘛还要加载一遍。

解决方法很简单，每次加载依赖模块前先看看依赖映射表是否存在，存在了即已加载过，就不需要重复加载了。

## 模块的内部方法是如何导出的

模块化开发的目的是为了避免污染全局变量，模块内部定义的方法，都是通过内部变量export的形式，提供外部访问句柄的。

```javascript
define('A', ['B','C'], function(require, exports) {
	exports.hello = function(name) {
		console.log('hello, ' + name)
	}
})

require('A', function(require) {
	var A = require('A')
	A.hello('answer')
})
```
上面代码表示模块A对外提供了一个`hello()`方法, 在模块D内部通过`require('A')`来取得模块A的内部对象exports,这样就可以使用hello方法啦。

我们先看下`var A = require('A')`都做了些什么？
```javascript
function require(id) {
	var mod = getMoulde(id)
	var fn = mod.factory;

	if(!mod.exports) {
        var exports = {};
	    var ret = factory.call(mod, new Require(), exports, mod);
	    if(ret) mod.exports = ret;

	    mod.exports = exports;
	}
	return mod.exports
}
```
利用call调用模块的factory方法,依次将`require`,`exports`对象传入,模块内部将hello方法挂在exports对象上,最后将exports对象挂在模块映射表上缓存起来，下次就可以直接使用了。


## 如何解决循环依赖

模块之间互相引用，会导致循环依赖：
```javascript
// a.js
S.declare(['cyclic/b'], function(require, exports) {
	
    var b = require('cyclic/b');
    exports.a = function() {
        return b;
    };

});

// b.js
S.declare(['cyclic/a'], function(require, exports) {

    var a = require('cyclic/a');
    exports.b = function() {
        return a;
    };

});
```
**备注：这里的循环依赖与重复加载是不同的概念**

重复加载指的是从网络下载模块文件，如何解决上面已经提到过了。
循环依赖的问题在于循环触发了模块的factory函数。解决的关键就是`require('xxx')`时需要判断是否factory已经执行过了，如果执行了，直接返回上次的exports对象即可。

# CMD,AMD区别

CMD推崇依赖就近，可以把依赖写进你的代码中的任意一行:
```javascript
define(function(require, exports, module) {
  var a = require('./a')
  a.doSomething()
  var b = require('./b')
  b.doSomething()
})
```
代码在运行时，首先是不知道依赖的，需要遍历所有的require关键字，找出后面的依赖。具体做法是将function toString后，用正则匹配出require关键字后面的依赖。显然，这是一种牺牲性能来换取更多开发便利的方法。

AMD是依赖前置，也就是在解析和执行当前模块之前，模块作者必须指明当前模块所依赖的模块:
```javascript
define(['./a','./b'],function(a,b){
   a.doSomething()
   b.doSomething()
}) 
```

由于提前知道依赖关系，所以不需要遍历整个函数体找它的依赖，所以性能相比CMD有所提升。唯一的缺点就是开发者必须显式得指明依赖——这会使得开发工作量变大

> 当你写到函数体内部几百上千行的时候，忽然发现需要增加一个依赖，你不得不回到函数顶端来将这个依赖添加进数组。

## 硬依赖?软依赖

所谓硬依赖就是肯定要执行的依赖，但有时候偏偏有这样的情况:
```javascript
// 函数体内：
if(status){
  a.doSomething()
}
```
我们把这种既可能依赖a,也可能不依赖a的这种叫做"软依赖"。为了最大化优化，我们需要针对这种情况做特殊处理:
```javascript
// 函数体内：
if(status){
  async(['a'],function(a){
    a.doSomething()
  })
}
```
这就是依赖前置+回调的方案。

* [AMD规范](https://github.com/amdjs/amdjs-api/wiki/AMD)


### 参考资料 

* 非 AMD 演示加载器地址 [webkit-dwarf](https://github.com/dwarfJS/webkit-dwarf)