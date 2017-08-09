---
title: 面试准备
date: 2017-07-29 16:09:22
tags: 面试准备
categories: 前端
---

# 简答题：
## settimeout 与 setInterval的区别， 及对他们的内存的分析
### 区别
1. setTimeout是在一段时间后调用指定函数（仅一次）
2. setInterval是每隔一段时间调用指定函数（N次）
```
function run(){
    // 其他代码
    setTimeout(function(){
        run();
    }, 10000);
}
run();
```
以上面的代码来说, 虽然设置的是10s执行一次, 但是实际时间却是需要// 其他代码的执行时间来确定
即setTimeout的间隔时间是, // setTimeout 的间隔时间 === 最小时间是(10s+)


```
setInterval(function(){
    run();
}, 10000);
```
而setInterval, 不会有上面的问题, 但是如果run()的执行时间, 操作大于10s, 那么甚至可能跳过任务;

[setInterval 和 setTimeout 会产生内存溢出](http://www.pjhome.net/article/Javascript/822.html)
[JavaScript setInterval()方法是否导致内存泄漏？](https://gxnotes.com/article/65338.html)

## 关于内存泄漏
### 内存
程序的运行需要内存。只要程序提出要求，操作系统或者运行时（runtime）就必须供给内存。
对于持续运行的服务进程（daemon），必须及时释放不再用到的内存。否则，内存占用越来越高，轻则影响系统性能，重则导致进程崩溃。
不再用到的内存，没有及时释放，就叫做内存泄漏（memory leak）。
（比如 C 语言）必须手动释放内存，程序员负责内存管理。
```
char * buffer;
buffer = (char*) malloc(42);

//...

free(buffer)    //手动释放内存
```

> 上面是 C 语言代码，malloc方法用来申请内存，使用完毕之后，必须自己用free方法释放内存。
这很麻烦，所以大多数语言提供自动内存管理，减轻程序员的负担，这被称为"垃圾回收机制"（garbage collector）。

### 垃圾回收机制
怎么知道哪些内存不再需要呢？常用的方法是 '引用计数', 语言的引擎有一张 '引用表', 保存了内存里面所有的资源(通常是各种值)的引用次数，当一个值的引用次数为 0 时，表示这个值用不到了，因此可将其释放。

但是如果一个值不再用到了，引用次数却不为 0 ，垃圾回收机制却无法释放这块内存，从而导致内存泄漏。
```
const arr = [1, 2, 3, 4];
console.log(arr);
```

打印完 arr 之后, arr 便用不到了，引用次数为 1, 但是它还会继续占用内存。
```
const arr = [1, 2, 3, 4];
console.log(arr);
arr = null;
```

arr 重置为 null，就解除了对 [1, 2, 3, 4] 的引用，引用次数变成了 0 ，内存就可以释放了。

### JavaScript 内存管理
JavaScript 是一种垃圾回收语言。垃圾回收语言通过周期性地检查先前分配的内存是否可达，帮助开发者管理内存。换言之，垃圾回收语言减轻了“内存仍可用”及“内存仍可达”的问题。两者的区别是微妙而重要的：仅有开发者了解哪些内存在将来仍会使用，而不可达内存通过算法确定和标记，适时被操作系统回收。

### JavaScript 内存泄漏

垃圾回收语言的内存泄漏主因是不需要的引用。理解它之前，还需了解垃圾回收语言如何辨别内存的可达与不可达。

**Mark-and-sweep**
1. 大部分垃圾回收语言用的算法称之为 Mark-and-sweep 。算法由以下几步组成：
垃圾回收器创建了一个“roots”列表。Roots 通常是代码中全局变量的引用。JavaScript 中，“window” 对象是一个全局变量，被当作 root 。window 对象总是存在，因此垃圾回收器可以检查它和它的所有子对象是否存在（即不是垃圾）；
2. 所有的 roots 被检查和标记为激活（即不是垃圾）。所有的子对象也被递归地检查。从 root 开始的所有对象如果是可达的，它就不被当作垃圾。
3. 所有未被标记的内存会被当做垃圾，收集器现在可以释放内存，归还给操作系统了。

现代的垃圾回收器改良了算法，但是本质是相同的：可达内存被标记，其余的被当作垃圾回收。

不需要的引用是指开发者明知内存引用不再需要，却由于某些原因，它仍被留在激活的 root 树中。在 JavaScript 中，不需要的引用是保留在代码中的变量，它不再需要，却指向一块本该被释放的内存。有些人认为这是开发者的错误。

为了理解 JavaScript 中最常见的内存泄漏，我们需要了解哪种方式的引用容易被遗忘。

#### 常见 JavaScript 内存泄漏
##### 意外的全局变量
JavaScript 处理未定义变量的方式比较宽松：未定义的变量会在全局对象创建一个新变量。在浏览器中，全局对象是 window 。
```
function foo(arg) {
    bar = "this is a hidden global variable";
}
```
真相是：
```
function foo(arg) {
    window.bar = "this is an explicit global variable";
}
```

函数 foo 内部忘记使用 var ，意外创建了一个全局变量。此例泄漏了一个简单的字符串，无伤大雅，但是有更糟的情况。

另一种意外的全局变量可能由 this 创建：
```
function foo() {
    this.variable = "potential accidental global";
}
// Foo 调用自己，this 指向了全局对象（window）
// 而不是 undefined
foo();
```
> 在 JavaScript 文件头部加上 'use strict'，可以避免此类错误发生。启用严格模式解析 JavaScript ，避免意外的全局变量。

全局变量注意事项:
尽管我们讨论了一些意外的全局变量，但是仍有一些明确的全局变量产生的垃圾。它们被定义为不可回收（除非定义为空或重新分配）。尤其当全局变量用于临时存储和处理大量信息时，需要多加小心。如果必须使用全局变量存储大量数据时，确保用完以后把它设置为 null 或者重新定义。与全局变量相关的增加内存消耗的一个主因是缓存。缓存数据是为了重用，缓存必须有一个大小上限才有用。高内存消耗导致缓存突破上限，因为缓存内容无法被回收。

##### 被遗忘的计时器或回调函数
在 JavaScript 中使用 setInterval 非常平常。一段常见的代码：
```
var someResource = getData();
setInterval(function() {
    var node = document.getElementById('Node');
    if(node) {
        // 处理 node 和 someResource
        node.innerHTML = JSON.stringify(someResource));
    }
}, 1000);
```

此例说明了什么：与节点或数据关联的计时器不再需要，*node* 对象可以删除，整个回调函数也不需要了。可是，计时器回调函数仍然没被回收（计时器停止才会被回收）。同时，*someResource* 如果存储了大量的数据，也是无法被回收的。

对于观察者的例子，一旦它们不再需要（或者关联的对象变成不可达），明确地移除它们非常重要。老的 IE 6 是无法处理循环引用的。如今，即使没有明确移除它们，一旦观察者对象变成不可达，大部分浏览器是可以回收观察者处理函数的。

观察者代码示例：
```
var element = document.getElementById('button');
function onClick(event) {
    element.innerHTML = 'text';
}
element.addEventListener('click', onClick);   // => 循环调用
```

**对象观察者和循环引用注意事项**
老版本的 IE 是无法检测 DOM 节点与 JavaScript 代码之间的循环引用，会导致内存泄漏。如今，现代的浏览器（包括 IE 和 Microsoft Edge）使用了更先进的垃圾回收算法，已经可以正确检测和处理循环引用了。换言之，回收节点内存时，不必非要调用 removeEventListener 了。

##### 脱离 DOM 的引用
有时，保存 DOM 节点内部数据结构很有用。假如你想快速更新表格的几行内容，把每一行 DOM 存成字典（JSON 键值对）或者数组很有意义。此时，同样的 DOM 元素存在两个引用：一个在 DOM 树中，另一个在字典中。将来你决定删除这些行时，需要把两个引用都清除.
```
var elements = {
    button: document.getElementById('button'),
    image: document.getElementById('image'),
    text: document.getElementById('text')
};
function doStuff() {
    image.src = 'http://some.url/image';
    button.click();
    console.log(text.innerHTML);
    // 更多逻辑
}
function removeButton() {
    // 按钮是 body 的后代元素
    document.body.removeChild(document.getElementById('button'));
    // 此时，仍旧存在一个全局的 #button 的引用
    // elements 字典。button 元素仍旧在内存中，不能被 GC 回收。
}
```
此外还要考虑 DOM 树内部或子节点的引用问题。假如你的 JavaScript 代码中保存了表格某一个 \<td> 的引用。将来决定删除整个表格的时候，直觉认为 GC 会回收除了已保存的 \<td> 以外的其它节点。实际情况并非如此：此 \<td> 是表格的子节点，子元素与父元素是引用关系。由于代码保留了 \<td> 的引用，导致整个表格仍待在内存中。保存 DOM 元素引用的时候，要小心谨慎。

##### 闭包
> 如果闭包的作用域中保存着一个 HTML 元素，则该元素无法被销毁。(下面代码来自高程)

闭包是 JavaScript 开发的一个关键方面：匿名函数可以访问父级作用域的变量。
```
function assgin() {
    var ele = document.getElementById('someEle');
    ele.onclick = function(){
        alert(ele.id);
    }
}
```

以上代码创建了一个作为 ele 元素事件处理程序的闭包，而这个闭包有创建了一个循环的引用，由于匿名函数保存了一个 assgin() 的活动对象的引用 ，因此无法减少对 ele 的引用次数 , 只要匿名函数存在，ele的引用次数至少是 1。我们可以稍微改写一下:
``` 
function assgin() {
    var ele = document.getElementById('someEle');
    var id = ele.id
    ele.onclick = function(){
        alert(id);
    }
    ele = null;
}
```

上面代码中，通过把 ele.id 的一个副本保存在一个变量中，并且在比保重引用该变量消除了循环引用，但是这样还不能解决内存泄露，*闭包会引用包含函数的整个活动对象*，而其中包含着 ele ，即使闭包不直接引用 ele ，包含函数的活动对象中也会保存 一个引用，因此需要把 ele 变量设置为 null ,这样就解除了对 DOM 对象的引用，减少其引用数，确保能正常回收。

关于内存的发现 chrome 的使用~暂时没有使用过，看不太明白，就不 copy 了。

[js闭包测试](http://www.cnblogs.com/rubylouvre/p/3345294.html) => 看不懂~

### 上述内容 copy 自下面二者：

[JavaScript 内存泄漏教程-阮一峰](http://www.ruanyifeng.com/blog/2017/04/memory-leak.html)
[4类 JavaScript 内存泄漏及如何避免](https://jinlong.github.io/2016/05/01/4-Types-of-Memory-Leaks-in-JavaScript-and-How-to-Get-Rid-Of-Them/)
## ajax 原生实现
```
var xhr = createXHR()
xhr.onreadystatechange = function() {
    if(xhr.readyState == 4) {
        if(xhr.status == 200) {
            console.log(xhr.responeText)
            //do sth...
        } else {
            console.log('request fail' + xhr.status)
        }
    }
};
xhr.open('get', 'hello.com', true)
xhr.send(null);
```

## 闭包的理解
闭包是指有权访问另一个函数作用域中的变量的函数。
这个口述我还是不知道怎么说，或许是应用不够~看了无数文章到头来敌不过忘记~也可能我理解的还是不到位吧~个人不解释了，放参考链接吧
[How do JavaScript closures work?--StackOverflow](https://stackoverflow.com/questions/111102/how-do-javascript-closures-work)
[学习Javascript闭包（Closure）--阮一峰的网络日志](http://www.ruanyifeng.com/blog/2009/08/learning_javascript_closures.html)
[JS 中的闭包是什么--方应杭](https://zhuanlan.zhihu.com/p/22486908)
[JavaScript 中 闭包 的详解](https://github.com/lin-xin/blog/issues/8)
[闭包--MDN](https://developer.mozilla.org/cn/docs/Web/JavaScript/Closures)
[闭包的应用](http://hexin.life/2017/04/15/title-7/)
## [html中一段文本内容 hdslakd*dnska8das ，将文本中含有数组['d', 'a', '*', '8'] 中的内容标记为红色文本(字符串有改动)](http://hexin.life/more/pdd.html)
### 设定 html 结构
```
    <style>
        .mark {
            color: red;
        }
    </style>
    
<html>
    <body>
        <div class='textToMark'>
        hdslakddnska8das
        </div>
    </body>
</html>
```

### 方法一:循环
```
    const textToMark = document.querySelector('.textToMark');
    
    const text = textToMark.innerHTML;

    const arr = ['d', 'a', '*', '8'];

    const newText = text.split('');

    function toMark (textArr, arr) {
        for(let i = 0; i < newText.length; i++) {
            for(let j = 0; j < arr.length; j++) {
                if(newText[i] == arr[j]) {
                    newText[i] = `<span class='mark'>${newText[i]}</span>`;
                }
            }
        }
        return newText;
    }
    toMark(newText, arr);
    textToMark.innerHTML = newText.join('');
```

### 方法二: 字符串的 replace
```
    const textToMark = document.querySelector('.textToMark');
    
    const text = textToMark.innerHTML;

    const reg = /[da\*8]+/g;

    var newtext = text.replace(reg, (match) => {
        return match = `<span class='mark'>${match}</span>`;
    });
    
    textToMark.innerHTML = newtext;
```

> 代码为个人写出，如果有更好的办法欢迎指教

## [原生JS创建这样的 dom 结构 < div id='hello'> < p class='textToMark'>hdslakddnska8das< p>< /div>](http://hexin.life/more/pdd.html)
```
function createElement() {
        var body = document.body;
    
        var fragment = document.createDocumentFragment()      

        var div = document.createElement('div')
        div.setAttribute('id', 'hello')

        fragment.appendChild(div)

        var p = document.createElement('p')
        p.className = 'textToMark'
        p.innerHTML = 'hdslakddnska8das'

        div.appendChild(p);
        body.appendChild(fragment)
    }
    createElement();
```

感谢评论指出，已改正，关于节点创建 createElement 的效率问题，如果**当插入的节点很多**的时候，createElement 的效率会不如 createDocumentFragment .
createElement 每次 append 一个节点的时候，都会导致页面的重排，例如:

数据为这样:
```
<ul id="myList">
    <li>
        <a href="www.baidu.com"></a>
    </li>
    <li>
        <a href="www.helloworld.com"></a>
    </li>
</ul>


var data = [
    { name: '36O秋招', url: 'http://campus.360.cn/2015/grad.html'},
    { name: 'TX校招', url: 'http://join.qq.com/index.php'}
]

```
```
function appendChildToElement(appendToElement, data) {
    var a, li;
    for (var i = 0, len = data.length; i < len; i++) {
        a = document.createElement('a');
        a.href = data[i].url;
        a.appChild(document.createTextNode(data[i].name))
        li = document.createElement('li');
        li.appendChild(a);
        appendChildToElement(li);
    }
}
```

这种情况下，data 内的每一个对象插入到 DOM 结构的时候都会触发一次重排，因此效率会较低。
但是我们可以改变他的 display 属性，临时从文档移除 ul ，即可有效减少重排次数。

```
var ul = document.getElementById('myList');
ur.style.display = 'none';
appendChildToElement(ul, data);
ul.style.display = 'block';
```

当然，更好的办法就是利用 createDocumentFragment 来创建一个文档片段.
```
var fragment = document.createElementFragment();
appendChildToElement(fragment, data);
document.getElementById('myList').appendChild(fragment);
```
只访问了一次 DOM 节点，只触发了一次重排;再次感谢 @xaclincoln 的指出。

查了一些关于 createDocumentFragment 和 createElement 比较的文章。
- [createDocumentFragment or createElement--StackOverflow](https://stackoverflow.com/questions/3397161/should-i-use-document-createdocumentfragment-or-document-createelement) 
- [createElement vs createDocumentFragment](https://jsperf.com/createelement-vs-createdocumentfragment)
- [createElement 与 createDocumentFragment 的点点区别](http://www.cnblogs.com/xesam/archive/2011/12/19/2293876.html)
- [CreateDocumentFragment 的用处](http://www.cnitblog.com/asfman/articles/32614.html)

## [创建一个函数对 JS 基础类型 ( function, boolean, array, number, string, object) 进行值复制](http://hexin.life/more/pdd.html)
```
    function valueToCopy (valueBeCopy) {
        var copyValue;
        if (typeof (+valueBeCopy) === 'number' && typeof valueBeCopy !== 'object') {
            copyValue = +valueBeCopy;
        } else if (typeof valueBeCopy === 'string') {
            copyValue = parseInt(copyValue);
        } else if (typeof valueBeCopy === 'object'){
            if(Array.isArray(valueBeCopy)) {
                copyValue = valueBeCopy.slice();
            }
            copyValue = JSON.parse(JSON.stringify(valueBeCopy))
        } 
            copyValue = valueBeCopy;
        // console.log(copyValue)
        return copyValue;   
    }
```

![test img](http://or3233yyd.bkt.clouddn.com//17-8-2/50845409.jpg)

## [url 输入到页面完成经历了什么](http://hexin.life/2017/08/10/title-23/)

[专门整理一篇尽可能详细的~](http://hexin.life/2017/08/10/title-23/)

# [选择题](http://hexin.life/more/pdd.html)
## 执行顺序
```
var input = document.getElementById('cls')

input.onmouseup = function() {
    console.log('onmouseup')
}
input.onmousedown = function() {
    console.log('onmousedown')
}
input.onclick = function() {
    console.log('onclick')
}
input.onfocus = function() {
    console.log('onfocus')
}
```

> onmousedown => onfocus => onmouseup => onclick

## [a 链接默认事件的阻止](http://hexin.life/more/pdd.html)
> A. a.onmouseup = function(e) {
        e.preventDefault()
    }
B.  a.onmousedown = function(e) {
        e.preventDefault()
    }
C.  a.onclick = function(e) {
        e.preventDefault()
     }
D. A B C 都可以~

 - => 经测试只有 onclick 可以    

## IE浏览器中 attachEvent 方式的事件绑定
> attachEvent的this总是Window。
```
el.attachEvent('onclick', function(){
    alert(this);
});
```

## HTTP状态码
- 400 Bad Request
由于明显的客户端错误（例如，格式错误的请求语法，太大的大小，无效的请求消息或欺骗性路由请求），服务器不能或不会处理该请求。[31]
- 401 Unauthorized（RFC 7235）
参见：HTTP基本认证、HTTP摘要认证
类似于403 Forbidden，401语义即“未认证”，即用户没有必要的凭据。[32]该状态码表示当前请求需要用户验证。

注意：当网站（通常是网站域名）禁止IP地址时，有些网站状态码显示的401，表示该特定地址被拒绝访问网站。
- 402 Payment Required
该状态码是为了将来可能的需求而预留的。该状态码最初的意图可能被用作某种形式的数字现金或在线支付方案的一部分，但几乎没有哪家服务商使用，而且这个状态码通常不被使用。如果特定开发人员已超过请求的每日限制，Google Developers API会使用此状态码。[34]
- 403 Forbidden
服务器已经理解请求，但是拒绝执行它。与401响应不同的是，身份验证并不能提供任何帮助，而且这个请求也不应该被重复提交。如果这不是一个HEAD请求，而且服务器希望能够讲清楚为何请求不能被执行，那么就应该在实体内描述拒绝的原因。当然服务器也可以返回一个404响应，假如它不希望让客户端获得任何信息。

## 选择正确答案(构造函数的引用地址)
```
var str = 'asd;  
var str2 = new String(str)  var str1 = new String(str)
console.log(str1 == str2 , str1 === str2)
```
A. true  true
B. true false
C. false true
D. false false

//  => 输出 => false false

> 因为 new 出来的俩个字符串引用地址不同

##  下面的输出结果 (this 指向问题)
```
    function one () { 
        this.name = 1;
        return function two () {
                name = 2;
            return function three() {
                var name = 3;
                console.log(this.name);
            }
        }
    }
    one()()()  // => 2;
```

> 还有一部分题忘掉喽 ~ 还有一些题具体的记不太清了，稍作修改，考点计本差不多，上面答案有的是我自己写的，有的是我 google 整理出来的，笔试期间摄像头坏了，而且不小心弹出去了三四次~就当练习了吧，反正简历也没准备好呢，哦，对了，考点大多都在高程中有详细讲解，需要好好看一下高程，面试应该会问一些 Node 和 ES6吧，如果有错误或者更好的方法请告诉我 

更多笔试整理更新在[个人博客](http://hexin.life/2017/08/01/title-22/)和[Github](https://github.com/18292843691/FE-interview)，欢迎小伙伴来一起准备秋招(求大腿抱)。


# 基础知识

## 使用 typeof 能得到哪些基础类型
- undefined
- boolean
- Number
- String
- Object          => null, object, array
- Function
## 何时使用 === 何时使用 ==

-  不需要判断类型是否相等，即需要隐式类型转换的时候使用 ==
判断对象中属性是否存在
```
if(obj.a == nul)  // => 等同于  obj.a === null || obj.a === undefined
// => jquery 推荐写法(源码)
```

- 其他情况全部用 ===

## 继承示例
示例一:
```
function GetArray(arr) {
    if(Array.isArray(arr)) {
        this.arr = arr
    } else if (typeof arr === 'string'){
        this.arr = arr.split('')
    } else {
        this.arr = [arr]
    }
}

GetArray.prototype.copy = function(newArr) {
    return this.arr = newArr.slice()
}

// 输入 
GetArray.prototype.delete = function(num, index, val) {
    if(val) {
        return this.arr.splice(findIndex((val)=> {
            return val === val
        }), num || 1)
    } else {
        return this.arr.splice(index, num || 1)
    }
}

var arr1 = [1,2,3,4]
var arr2 = []
var arr = new GetArray(arr1)

arr.copy(arr2)
arr.delete(1, 0)
```

示例二: 
```
function Person (name, age) {
    this.name = name || 'just people'
    this.age = age || 'forwver'
}

Person.prototype.skill = function(val) {
    console.log('i can ' + val)
}

function Man (name, age, sex) {
    this.name = this.name
    this.age = this.age
    this.sex = 'man'
}

Person.prototype.love = function(name) {
    if(name) {
        console.log(this.name + ' love ' + name)
    } else {
        console.log(this.name + 'love self')
    }
}

function Men (name, age, sex) {
    this.name = name || this.name
    this.age = age || this.age
    this.sex = 'men'
}

Man.prototype = new Person()

Men.prototype = new Person()

function Boy (name, age, sex) {
    this.name = name
    this.age = age
    this.sex = this.sex
}

Boy.prototype = new Man('man')

function Girl (name, age, sex) {
    this.name = name
    this.age = age
    this.sex = this.sex
}

Girl.prototype = new Men('girl')

var hx = new Boy('hx', 22)
var xbk = new Girl('xbk', 20)
var h = new Man('fh',0)
```

##window.onload 和 DOMContentLoaded 的区别

###window.onload

- window.onload 需要等到全部的资源加载完成才会触发，不能够编写多个，编写多个时只会执行最后一个,该方法可以绑定到任意元素上

```
<script>
        window.onload = function() {
            console.log('DOM READY', 1)
        }
        window.onload = function() {
            console.log('DOM READY', 2)
        }
        window.onload = function() {
                console.log('DOM READY', 3)
            }
            //DOM READY 3
    </script>
```

### DOMContentLoaded 

- (HTML5) DOMContentLoaded 类似于 jQuery 的$(document).ready(),页面文档（DOM）完全加载并解析完毕之后触发，不会等图片，iframe, css 等资源加载完成。 如果多次使用，会按顺序依次执行相应任务

```javascript
<body>
    <script>
        document.addEventListener("DOMContentLoaded", function (event) {
            alert("DOM ready1");
        });
        document.addEventListener("DOMContentLoaded", function (event) {
            alert("DOM ready2");
        });
    </script>
    <script>
        document.addEventListener("DOMContentLoaded", function (event) {
            alert("DOM ready3");
        });
        document.addEventListener("DOMContentLoaded", function (event) {
            alert("DOM ready4");
        });
// DOM ready1
// DOM ready2
// DOM ready3
// DOM ready4
    </script>
</body>

```

> H5之前使用document.onreadystatechange 事件并手动判断document.readyState == 'complete'   来代替。

### document.readyState / readtstatechange 

用来表示 DOM 的加载状态，该属性值发生变化时会触发 readtstatechange 事件。有三种取值：
 - loading:    DOM 加载中
 - interactive:    DOM 就绪但资源仍在加载中
 - compelte :    DOM 加载完成

由于 IE8 支持 document.readState 属性,因此在 IE8 中常常用来做 DOMContentLoaded 的降级 (fallback)

 >  注意IE8以前的IE不支持document.readyState属性。 可以执行 document.documentElement.doScroll("left")， 当DOM未就绪时执行该方法会抛出错误，以此检测DOM是否就绪。
### jQuery

jQuery也提供了三种方法载入页面事件
- $(document).ready(callback)：DOM就绪时执行回调函数，返回值为 docuument 的jQuery集合
- $(function() {})：常用写法，返回和参数同上
- $(window).load()：当 DOM 完全加载完毕时，页面渲染完成即图片等资源加载完成时触发
> [.ready() 的实现](https://github.com/jquery/jquery/blob/master/src/core/ready.js)

```javascript
if ( document.readyState === "complete" ||
    ( document.readyState !== "loading" && !document.documentElement.doScroll ) ) {
    // Handle it asynchronously to allow scripts the opportunity to delay ready
    window.setTimeout( jQuery.ready );
} else {
    // Use the handy event callback
    document.addEventListener( "DOMContentLoaded", completed );
    // A fallback to window.onload, that will always work
    window.addEventListener( "load", completed );
}
```

### 参考 

- [兼容所有浏览器的 DOM 载入事件](http://harttle.com/2016/05/14/binding-document-ready-event.html)

## [用 JS创建10个 a 标签点击弹出对应序号](http://hexin.life/more/js-test-1.html)

```javascript
(function() {
            let body = document.body;
            for (let i = 0; i < 10; i++) {
                let a = document.createElement('a');

                a.textContent = `${i}`;
                a.style.cssText = 'margin: 5px;' + 'color: #363636;' + 'font-size: 16px';

                body.appendChild(a);
            }

            let ac = document.getElementsByTagName('a');
            //console.log(ac)
            for (let i = 0; i < 10; i++) {

                ac[i].onclick = function() {
                        alert(i)
                    }
                    // ac[i].addEventListener('click', function() {
                    //     alert(ac[i].textContent)
                    // })
            }
        })()
```

## 下面代码执行结果

### 1. (预解析)

#### 1.1 下面代码执行结果

```
console.log( func );
var func = 123;
    console.log( func );
function func() {
  console.log( 'Hello js' );
}

// => 第一个 console.log(func) => 打印出下面 func 这个函数体
// => 第二个 console.log(func) => 打印出刚刚定义的 func 123
            //ƒ func() {
            //  console.log( 'Hello js' );
            //}
            //123

/*********************/

console.log( func() );
var func = 123;
console.log( func );
function func() {
  console.log( 'Hello js' );
}
// => 第一个则打印函数内部的出 hello js
// => 然后打印出自身的 undefined
// => 第二个 console.log(func) 则打印出 123
```

#### 1.2 下面代码执行结果

```
function func2 () {
  console.log('1', num );
  var num = 456;       // => var num 会提前
  console.log('2', num );
}
var num = 123;
func2(num);
console.log('3',num)
// 1 undefined
// 2 456
// 3 123
```

#### 1.3 下面代码执行结果

```
function func3 () {
  console.log('1', num );
  num = 456;         //=>
  console.log('2', num );
}
var num = 123;
func3( num );
console.log( '3',num );
// 1 123
// 2 456
// 3 456
```

#### 1.4下面代码执行结果

```
var num = 123;
function f1 () {
  console.log( num );  
}
function f2 () {
  var num = 456;
  f1();
}
f2();
console.log( num );
// 123
// 123
// => 调用 f2() => 进入 f2() 内部，var num 被提前解析, f2() 内部执行 num =
// 456 => 进入 f1() => 这里找不到 num 值, 向全局查询， 发现 num = 123; 打印
// => 最后一个打印全局 num = 123;
```

### 1. 下面代码执行结果

```
var length = 10;
function fn() {
  console.log( this.length )
}
var obj = {
  length: 5,
  mathod: function( fn ){
    fn();
    arguments[ 0 ]();
  }
}
obj.mathod( fn, 1 );
// 10 2
```

> // =>  这里打印出了 10 和 2
// => 10 好理解， 这里难理解的是 2 .
// => 第一个 10 是因为 this 指向了 window， 第二个则是指向了函数内部，打印出来的是函数的长度 

![](http://or3233yyd.bkt.clouddn.com//17-7-31/38769321.jpg) 
如果将函数的 length 改为 len 则很明显了

```
var len = 10;
function fn() {
    
  console.log( this.len )
}
var obj = {
  len: 5,
  mathod: function(){
    fn();
    arguments[ 0 ]();
  }
}
obj.mathod( fn , 2);
// 10 undefined
```

![](http://or3233yyd.bkt.clouddn.com//17-7-31/63463663.jpg)

> 我只能理解到这里了，如果有什么错误，希望大神指教。

整理来自：
- [整理前端面试题(二):预解析及作用域 (含美团面试题)](http://www.jianshu.com/p/94ee757ca30d)

# [HTTP](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Overview)
## HTTP response codes
- [HTTP response codes](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status)
- [HTTP状态码](https://zh.wikipedia.org/wiki/HTTP%E7%8A%B6%E6%80%81%E7%A0%81)
- [你所知道的3xx状态码](https://aotu.io/notes/2016/01/28/3xx-of-http-status/)
## HTTP 2.0 
- HTTP2.0性能增强的核心：二进制分帧
- HTTP2.0 首部压缩
- 所有的HTTP2.0的请求都在一个TCP链接上
- 并行双向字节流的请求和响应
- HTTP2.0的请求优先级
- HTTP2.0的服务器推送

[TAT.bizaiHTTP2.0的奇妙日常](http://www.alloyteam.com/2015/03/http2-0-di-qi-miao-ri-chang/)
[TAT.tennylvHTTP,HTTP2.0,SPDY,HTTPS你应该知道的一些事](http://www.alloyteam.com/2016/07/httphttp2-0spdyhttps-reading-this-is-enough/)
[HTTP2简介和基于HTTP2的Web优化](https://github.com/creeperyang/blog/issues/23)
[HTTP/2-维基百科](https://zh.wikipedia.org/wiki/HTTP/2)

## HTTPS

[HTTPS 与 SSL 证书概要](http://www.runoob.com/w3cnote/https-ssl-intro.html)
[淘宝全站HTTPS实践](http://velocity.oreilly.com.cn/2015/ppts/lizhenyu.pdf)

##[HTTP协议中的短轮询、长轮询、长连接和短连接](http://www.cnblogs.com/tester-l/p/6018114.html)
## [使用 HTTP 缓存：Etag, Last-Modified 与 Cache-Control](http://harttle.com/2017/04/04/using-http-cache.html)

> 这篇文章中的 Cache-Control 中的 no-cache 有误，no-cache 不是不可缓存的意思

- “no-cache”表示必须先与服务器确认返回的响应是否发生了变化，然后才能使用该响应来满足后续对同一网址的请求。因此，如果存在合适的验证令牌 (ETag)，no-cache 会发起往返通信来验证缓存的响应，但如果资源未发生变化，则可避免下载。
- 相比之下，“no-store”则要简单得多。它直接禁止浏览器以及所有中间缓存存储任何版本的返回响应，例如，包含个人隐私数据或银行业务数据的响应。每次用户请求该资产时，都会向服务器发送请求，并下载完整的响应。

[详情见下方](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn)
## HTTP 缓存
[HTTP 缓存](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn)
[HTTP 缓存-MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ)

# 简述如何实现一个模块加载器，实现类似 requirejs 的基本功能

# JavaScript

## 实现数组的随机排序
```javascript
// 数组随机排序
            var arr = [11, 22, 33, 44, 55, 66, 77, 88, 99, 100];

            let len = arr.length;

            function RandomArray(arr) {
                if (arr.length == 1) {
                    return arr;
                }
                for (let i = 0; i < len; i++) {
                    let random = Math.floor(Math.random() * len);
                    arr.push(arr[random]);
                    arr.splice(random, 1);
                }
                console.log(arr);
                return arr;
            }

            RandomArray(arr);
```
更多实现：
- [数组元素随机化排序算法实现](http://div.io/topic/1610)
- [数组随机排序](https://www.w3cplus.com/javascript/how-to-randomize-shuffle-a-javascript-array.html)
- [Fisher–Yates shuffle 洗牌算法](https://gaohaoyang.github.io/2016/10/16/shuffle-algorithm/#top)

## 实现数组的去重

### ES5
```
var arr = [1, '1', 1, 2, 3, 2, 4, '5', 5];

function duplicatesWithString(arr) {
    var newArr = [];
    var tmpArr = [];
    arr.forEach(function(val) {
        var key = (typeof val) + val;
        if(!tmpArr[key]) {            //如果为 undefined 就进入循环
            tmpArr[key] = true;       //设置为 true 表示已经存在该数字
            newArr.push(val)
        }
    })
    return newArr;
}

duplicatesWithString(arr);    // =>  [1, "1", 2, 3, 4, "5", 5]

/******************************/

//如果想去掉字符串
function duplicatesNoString(arr) {
    var newArr = [];
    var tmpArr = [];
    arr.forEach(function(val) {
        if(typeof val !== 'string') {
            if(!tmpArr[val]) {            //如果为 undefined 就进入循环
                tmpArr[val] = true;       //设置为 true 表示已经存在该数字
                newArr.push(val);      
            }
        }       
    })
    return newArr;
}
duplicatesNoString(arr)
```

### ES6
ES6中新增 set 数据结构

> set 中不能添加重复元素，如：

```
var mySet = new Set();

mySet.add(1); // Set { 1 }
mySet.add(5); // Set { 1, 5 }
mySet.add(5); // Set { 1, 5 }
mySet.add('some text'); // Set { 1, 5, 'some text' }
```

1. 因此我们可利用这个特性

```
let arr1 = [1, '1', 1, 2, 3, 2, 4, '5', 5];
let set = new Set(arr1);
console.log(set) // =>{1, "1", 2, 3, 4,"5", 5}
```

2. ES6中Array新增了一个静态方法Array.from，可以把类似数组的对象转换为数组(如通过 QuerySelectAll 得到的 Node List)

```
let arr2 = [1, '1', 1, 2, 3, 2, 4, '5', 5];
let array = Array.from(new Set(arr2));
console.log(array) // =>[1, "1", 2, 3, 4, "5", 5]
```
### 参考
[六种方法数组去重](http://blog.csdn.net/q1056843325/article/details/73277063)
[一行代码实现数组去重（ES6）](http://blog.csser.me/posts/E4B880E8A18CE4BBA3E7A081E5AE9EE78EB0E695B0E7BB84E58EBBE9878DEFBC88ES6EFBC89)
## 实现一个函数，输入 123456789 输出 123，465，789
```
function cal(arr) {
        var newArr = [];
        var arr = typeof arr === 'string' ? arr : arr.toString();
        arr = arr.split('');
        arr.forEach(function(value, index) {
            if(index % 3 === 0 && index != 0) {
                newArr.push(',')
            }
            newArr.push(arr[index]);
        })
        return newArr.join('');
    }
// 判断为三则push进新的数组
```
## JavaScript 中 apply 、call、bind 
[JavaScript 中 apply 、call 的详解](http://blog.gdfengshuo.com/2017/03/21/6/)
[js的call() ，apply() 两种方法的区别和用法，最白话文的解释，让枯燥滚粗！](http://blog.csdn.net/xllily_11/article/details/51480723)
## JS 实现不用第三个变量交换俩个变量
我自己想的是前段时间学习 ES6 时遇到的；
> [a, b] = [b, a]

然后网上搜了一下，发现有人整理的很全，就拉过来学习吧

[ 七种方案解决JavaScript交换两个变量值的问题](http://blog.csdn.net/q1056843325/article/details/53223914)

## 实现对对象和数组的 forEach

```
function forEach(obj, index, value, fn) {
    if (obj == null) return;
    if (!fn) {
        if (typeof obj === 'function' && obj.call) {
            fn = Function
        } else if (typeof obj.forEach === 'function' && obj.forEach != arguments.callee) {
            // 如果对象内部已经实现 forEach ，例如数组
            obj.forEach(index, value)
            return;
        } else if (typeof obj.length == 'number') {
            // 如果是数组或者类数组的对象
            _Array_forEach(obj, index, value)
            return;
        }
    }
    _Function_forEach(fn || Object, obj, index, value)
}


function _Array_forEach(arr, index, value) {
    if(arr == null) return ;
    let i = 0,
        len = arr.length;
    if(typeof arr === 'string') {
        for(; i < len; i++) {
            index.call(value, arr.charAt(i), i, arr)
        }
    } else {
        for(; i < len; i++) {
            index.call(value, arr[i], i, arr)
        }
    }
}


function _Function_forEach (fn, obj, index, value) {
    for(let key in obj) {
        if(obj.hasOwnProperty(key))
            index.call(value, obj[key], key, obj)
    }
}
```

采自 大神司徒正美[javascript 的forEach函数](http://www.cnblogs.com/rubylouvre/archive/2009/11/10/1599978.html)

## 实现对时间 xxxx-xx-xx 的获取

```
function formatDate(date) {
    if (!date) {
        date = new Date()
    }

    const year = date.getFullYear(),
        month = (date.getMonth() + 1),         // month 是从 0 开始
        day = date.getDate(),
        hours = date.getHours(),
        minutes = date.getMinutes(),
        seconds = date.getSeconds();


    return year + '-' + month + '-' + day
}

var date = new Date()
var formatDate = formatDate(date)
console.log(formatDate)
2017-8-8
```


# HTML 知识
## [用 div 实现 textarea](http://hexin.life/more/用div制作textarea.html)

> Textarea 对象代表 HTML 表单中的一个文本域 (text-area)。

1. 首先我们需要可输入功能
  -  HTML5 新增 [contenteditable](http://www.runoob.com/tags/att-global-contenteditable.html) 属性
他有 true 和 flase 俩个值，true  指定元素是可编辑的

```
<div contenteditable='true'， class='div-text'>
        hello
</div>
```

此时效果已经有了，点击是自带聚焦发光还挺好看的，但是只有点击上的时候才会显示可输入框，我们加一个 border 模拟一下即可。
```css
.div-text{
    min-height: 20px;
    min-width: 80px;
}

.div-text{
    padding-left: 3px;
    width: 100px;
    border: 1px solid #ccc;
}
```

然后是自定义拖动大小, 利用 CSS3 的 resize ，再增加 overflow 属性
```css
	margin: 50px;
	padding-left: 3px;
	width: 100px;
	border: 1px solid #ccc;
	resize: both;
	overflow: hidden;
```

![测试图片](http://or3233yyd.bkt.clouddn.com//17-8-1/97830271.jpg)

> 不喜欢 textarea 右下角图标时，也可以利用这个属性去除右下角的拖动图标

```css
resize: none;
```

## canvas、svg的区别
> 由于我没做过 svg ，canvas 也只是联系国一个时钟而已，就在这里搜了一些资料。

- [SVG 与 HTML5 的 canvas 各有什么优点，哪个更有前途？](https://www.zhihu.com/question/19690014)
- [Canvas 和 SVG 都允许您在浏览器中创建图形，但是它们在根本上是不同的-w3school](http://www.w3school.com.cn/html5/html_5_canvas_vs_svg.asp)
- [有了css3之后，我们还需要canvas还有svg吗？](https://segmentfault.com/q/1010000000459364)
- [Canvas 与 SVG的主要区别 #13-github](https://github.com/abcrun/abcrun.github.com/issues/13)
- [SVG初体验](https://segmentfault.com/a/1190000005991537)
 
## 可访问性
### [如何提高网站的可访问性（Accessibility）？——来自W3C的指南](https://zhuanlan.zhihu.com/p/23873058)
### [10步大幅提升网站可访问性](http://isux.tencent.com/ten-steps-enhance-web-accessibility.html)

# [CSS 知识](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference)
## 链接的伪类顺序（:link :visited :hover :active）
[CSS中超链接伪类link,visited,hover,active的顺序分析](http://www.dengzhr.com/frontend/css/344)
## CSS中link 和@import的区别是
- 差别1：老祖宗的差别。link属于XHTML标签，而@import完全是CSS提供的一种方式。

link标签除了可以加载CSS外，还可以做很多其它的事情，比如定义RSS，定义rel连接属性等，@import就只能加载CSS了。

- 差别2：加载顺序的差别。当一个页面被加载的时候（就是被浏览者浏览的时候），link引用的CSS会同时被加载，而@import引用的CSS会等到页面全部被下载完再被加载。所以有时候浏览@import加载CSS的页面时开始会没有样式（就是闪烁），网速慢的时候还挺明显（梦之都加载CSS的方式就是使用@import，我一边下载一边浏览梦之都网页时，就会出现上述问题）。

- 差别3：兼容性的差别。由于@import是CSS2.1提出的所以老的浏览器不支持，@import只有在IE5以上的才能识别，而link标签无此问题。

- 差别4：使用dom控制样式时的差别。当使用javascript控制dom去改变样式的时候，只能使用link标签，因为@import不是dom可以控制的。

大致就这几种差别了（如果还有什么差别，大家告诉我，我再补充上去），其它的都一样，从上面的分析来看，还是使用link标签比较好。

- 差别5：@import可以在css中再次引入其他样式表，比如可以创建一个主样式表，在主样式表中再引入其他的样式表，如：

```css
main.css
———————-
@import “sub1.css”;
@import “sub2.css”;

sub1.css
———————-
p {color:red};

sub2.css
———————-
.myclass {color:blue}

```
> 这样更利于修改和扩展
(*但是这样做有一个缺点，会对网站服务器产生过多的HTTP请求，以前是一个文件，而现在却是两个或更多文件了，服务器的压力增大，浏览量大的网站还是谨慎使用。有兴趣的可以观察一下像新浪等网站的首页或栏目首页代码，他们总会把css或js直接写在html里，而不用外部文件*)

[详解CSS外部引用之link与@import的区别-大前端](http://www.daqianduan.com/2417.html)
[外部引用CSS中 link与@import的区别](http://www.dreamdu.com/blog/2007/05/11/css_link_import/) v
## CSS 选择符有哪些？哪些属性可以继承？优先级算法如何计算？ CSS3新增伪类有那些？CSS3新增了哪些特性？

### CSS 选择符：
 - id 选择器(#id): HTML元素以id属性来设置id选择器,CSS 中 id 选择器以 "#" 来定义。
 - class 选择器(.class): class 选择器在HTML中以class属性表示, 在 CSS 中，类选择器以一个点"."号显示：
 - 标签选择器(div / p /span):  在 css 中直接使用 html 的标签进行选择
 - 相邻选择器(div1,div2,div3):  选择所有 div1，div2，div3 元素
 - 子选择器(div>p): 选择父元素为 div 元素的所有 p 元素
 - 后代选择器(div p): 选择 div 元素内部的所有 p 元素。
 - 通配符选择器( * ): 选择所有元素。
 - 属性选择器([target]): 选择所有属性为 target 的元素。
  - [target=_blank]: 选择 target="_blank" 的所有元素。
  - [title~=flower]: 选择 title 属性包含单词 "flower" 的所有元素。
  - [lang|=en]: 选择 lang 属性值以 "en" 开头的所有元素。
  - :link	(a:link)	选择所有未被访问的链接。	1

选择器	      例子	       例子描述          	CSS
- :visited	(a:visited)	选择所有已被访问的链接。	1
- :active	(a:active)	选择活动链接。	1
- :hover	(a:hover)	选择鼠标指针位于其上的链接。	1
- :focus	(input:focus)	选择获得焦点的 input 元素。	2
- :first-letter	(p:first-letter)	选择每个 p 元素的首字母。	1
- :first-line	(p:first-line)	选择每个 p 元素的首行。	1
- :first-child	(p:first-child)	选择属于父元素的第一个子元素的每个 p 元素。	2
- :before	(p:before)	在每个 p 元素的内容之前插入内容。	2
- :after	(p:after)	在每个 p 元素的内容之后插入内容。	2

等等等等....查看手册吧~太多了，记住常用的就行了，其他的用到了再查吧。
#### CSS 伪类(觉得这个挺重要的-列举的有点多)
- :first-of-type	(p:first-of-type)	选择属于其父元素的首个 p 元素的每个 p 元素。	3
- :last-of-type	(p:last-of-type)	选择属于其父元素的最后 p 元素的每个 p 元素。	3
- :only-of-type	(p:only-of-type)	选择属于其父元素唯一的 p 元素的每个 p 元素。	3
- :only-child	(p:only-child)	选择属于其父元素的唯一子元素的每个 p 元素。	3
- :nth-child(n)	(p:nth-child(2))	选择属于其父元素的第二个子元素的每个 p 元素。	3
- :nth-last-child(n)	(p:nth-last-child(2))	同上，从最后一个子元素开始计数。	3
- :nth-of-type(n)	(p:nth-of-type(2))	选择属于其父元素第二个 p 元素的每个 p 元素。	3
- :nth-last-of-type(n)	(p:nth-last-of-type(2))	同上，但是从最后一个子元素开始计数。	3
- :last-child	(p:last-child)	选择属于其父元素最后一个子元素每个 p 元素。	3
- :last-child	p:last-child	选择属于其父元素最后一个子元素每个 p 元素。	3
- :active	向被激活的元素添加样式。	1
- :focus	向拥有键盘输入焦点的元素添加样式。	2
- :hover	当鼠标悬浮在元素上方时，向元素添加样式。	1
- :link	向未被访问的链接添加样式。	1
- :visited	向已被访问的链接添加样式。	1
- :first-child	向元素的第一个子元素添加样式。	2
- :lang	向带有指定 lang 属性的元素添加样式。	2
- :root	:root	选择文档的根元素。	3
- :empty	( p:empty )	选择没有子元素的每个 p 元素（包括文本节点）。	3
- :target	( #news:target )	选择当前活动的 #news 元素。	3
- :enabled	( input:enabled )	选择每个启用的 input 元素。	3
- :disabled	( input:disabled )	选择每个禁用的 input 元素	3
- :checked	( input:checked )	选择每个被选中的 input 元素。	3
- :not(selector)	( :not(p) )	选择非 p 元素的每个元素。	3
- ::selection	( ::selection )	选择被用户选取的元素部分。	3
 
##### 部分应用
###### :after伪类 

- 经典的[清除浮动](http://hexin.life/2017/07/29/title-22/) => :[after伪类+content 清除浮动的影响](http://www.zhangxinxu.com/wordpress/2010/09/after%E4%BC%AA%E7%B1%BBcontent%E5%86%85%E5%AE%B9%E7%94%9F%E6%88%90%E5%B8%B8%E8%A7%81%E5%BA%94%E7%94%A8%E4%B8%BE%E4%BE%8B/)
```
.fix:after{
	display:block; 
	content:"clear"; 
	height:0; 
	clear:both; 
	overflow:hidden; 
	visibility:hidden;
}
.fix{*zoom:1;}  /* IE */
```

- [:after伪类+content 让大小不固定图片垂直居中](http://www.zhangxinxu.com/wordpress/2010/09/after%E4%BC%AA%E7%B1%BBcontent%E5%86%85%E5%AE%B9%E7%94%9F%E6%88%90%E5%B8%B8%E8%A7%81%E5%BA%94%E7%94%A8%E4%B8%BE%E4%BE%8B/)

```
.pic_box{
	width:300px; 
	height:300px; 
	background-color:#beceeb; 
	font-size:0; 
	*font-size:200px; 
	text-align:center;
	}
.pic_box img{
	vertical-align:middle;
}
.pic_box:after{
	display:inline-block; 
	width:0; 
	height:100%; 
	content:"center"; 
	vertical-align:middle; 
	overflow:hidden;
}
```

###### ::selection 伪元素(css3 唯一新增伪元素)
> 浏览器自带选取后的背景是蓝色，我们可以利用这个将选取背景设定称自己喜欢的样子
```html
<p>dsadkladja</p>
```

```css
p::selection{
	background-color: red;
}
```

效果： ![](http://or3233yyd.bkt.clouddn.com//17-8-1/35733865.jpg)

[CSS 选择器参考手册-w3school](http://www.w3school.com.cn/cssref/css_selectors.ASP)

#### 到这里已经分不清什么是伪类什么是伪元素了
伪类:
- 用于当已有元素处于的某个状态时，为其添加对应的样式，这个状态是根据用户行为而动态变化的。比如说，当用户悬停在指定的元素时，我们可以通过:hover来描述这个元素的状态。虽然它和普通的css类相似，可以为已有的元素添加样式，但是它只有处于dom树无法描述的状态下才能为元素添加样式，所以将其称为伪类。

![](http://www.alloyteam.com/wp-content/uploads/2016/05/%E4%BC%AA%E7%B1%BB.png)

伪元素：
- 用于创建一些不在文档树中的元素，并为其添加样式。比如说，我们可以通过:before来在一个元素前增加一些文本，并为这些文本添加样式。虽然用户可以看到这些文本，但是这些文本实际上不在文档树中。
*虽然CSS3标准要求伪元素使用双冒号的写法，但也依然支持单冒号的写法。为了向后兼容，我们建议你在目前还是使用单冒号的写法。*
![](http://www.alloyteam.com/wp-content/uploads/2016/05/%E4%BC%AA%E5%85%83%E7%B4%A0.png)

> 伪类的操作对象是文档树中已有的元素，而伪元素则创建了一个文档数外的元素。因此，伪类与伪元素的区别在于：有没有创建一个文档树之外的元素。
[总结伪类与伪元素-腾讯 AlloyTeam](http://www.alloyteam.com/2016/05/summary-of-pseudo-classes-and-pseudo-elements/)
[CSS伪类与CSS伪元素的区别及由来](https://swordair.com/origin-and-difference-between-css-pseudo-classes-and-pseudo-elements/)
[CSS3伪类与伪元素的区别及注意事项](http://blog.csdn.net/q1056843325/article/details/53560588)
### CSS 可继承属性
- 不可继承的：display、margin、border、padding、background、height、min-height、max-height、width、min-width、max-width、overflow、position、left、right、top、bottom、z-index、float、clear、table-layout、vertical-align、page-break-after、page-bread-before和unicode-bidi。
- 所有元素可继承：visibility和cursor。
- 内联元素可继承：letter-spacing、word-spacing、white-space、line-height、color、font、font-family、font-size、font-style、font-variant、font-weight、text-decoration、text-transform、direction。
- 终端块状元素可继承：text-indent和text-align。
- 列表元素可继承：list-style、list-style-type、list-style-position、list-style-image。
- 表格元素可继承：border-collapse。

> 个人不太喜欢记这些，记一下常用的吧~其他的慢慢经验积累吧，一样铺链接

[Which CSS properties are inherited?](https://stackoverflow.com/questions/5612302/which-css-properties-are-inherited)
[Appendix F. Full property table-其中列出了是否可以继承](https://www.w3.org/TR/CSS21/propidx.html)

### CSS3 权重

> !important > 内联 > ID > 类 > 标签 | 伪类 | 属性选择 > 伪对象 > 通配符 > 继承

#### 选择器权重值的计算 A B C D
> 内联 => id => 伪类 => 伪元素及标签  
>  0 => 0 => 0 => 0

- A：如果规则是写在标签的style属性中（内联样式），则A=1，否则，A=0. 对于内联样式，由于没有选择器，所以 B、C、D 的值都为 0，即 A=1, B=0, C=0, D=0（简写为 1,0,0,0，下同）。
- B：计算该选择器中ID的数量。（例如，#header 这样的选择器，计算为 0, 1, 0, 0）。
- C：计算该选择器中伪类及其它属性的数量（包括类选择器、属性选择器等，不包括伪元素）。 （例如， .logo[id='site-logo'] 这样的选择器，计算为 0, 0, 2, 0）。
- D：计算该选择器中伪元素及标签的数量。（例如，p:first-letter 这样的选择器，计算为0, 0, 0, 2）。

计算权重值时，A, B, C, D 四组值，从左到右，分组比较，如果 A 相同，比较 B, 如果 B 相同，比较 C, 如果 C 相同，比较 D, 如果 D 相同，后定义的优先。

#### 关于 inherit 来的属性

> 继承而来的属性值，权重永远低于明确指定到元素的定义。只有当一个元素的某个属性没有被直接指定时，才会继承父级元素的值

#### 特殊的 !important

> 有 !important 指定的权重值大于所有未用 !important 指定的规则。包括内联

*不过我记得 chrome 浏览器自带的 input框聚焦时的黄色 好像是 !important 都不能覆盖的*
如果多条规则中都对同一个属性指定了 !important 呢？这时候 !important 的作用相互抵销，依然按照ABCD四组计算比较。
[如何和何时使用CSS的!important](https://www.w3cplus.com/css/the-important-css-declaration-how-and-when-to-use-it.html)

[深入解析CSS样式层叠权重值](https://ofcss.com/2011/05/26/css-cascade-specificity.html)

[CSS选择器距离无关](https://swordair.com/css-selectors-independent-of-the-distance/)

## CSS 清除浮动
### [clearfix改良及overflow:hidden详解【译文】](http://www.iyunlu.com/view/css-xhtml/56.html)
### [那些年我们一起清除过的浮动](http://www.iyunlu.com/view/css-xhtml/55.html)
### [CSS之BFC详解](http://www.html-js.com/article/1866)
### [CSS float浮动的深入研究、详解及拓展(一)](http://www.zhangxinxu.com/wordpress/2010/01/css-float%E6%B5%AE%E5%8A%A8%E7%9A%84%E6%B7%B1%E5%85%A5%E7%A0%94%E7%A9%B6%E3%80%81%E8%AF%A6%E8%A7%A3%E5%8F%8A%E6%8B%93%E5%B1%95%E4%B8%80/)
### [css-float浮动的深入研究、详解及拓展二](http://www.zhangxinxu.com/wordpress/2010/01/css-float%E6%B5%AE%E5%8A%A8%E7%9A%84%E6%B7%B1%E5%85%A5%E7%A0%94%E7%A9%B6%E3%80%81%E8%AF%A6%E8%A7%A3%E5%8F%8A%E6%8B%93%E5%B1%95%E4%BA%8C/)
## CSS3新增特性
- CSS3 边框
 - border-radius
 - box-shadow
 - [border-image](http://www.runoob.com/try/try.php?filename=trycss3_border-image) => 
      border-image: source slice width outset repeat; => border-image: url(border.png) 30 round ( 设置图像边界是否应重复（repeated）、拉伸（stretched）或铺满（rounded）);
- CSS3 圆角 => border-radius
- CSS3 背景
 - background-image
 - [background-size](http://www.runoob.com/try/try.php?filename=trycss3_background-size)
 - [background-origin](http://www.runoob.com/try/try.php?filename=trycss3_background-origin) => background-origin:content-box ( border-box );
 - background-clip 
- [CSS3 渐变 ](http://www.runoob.com/css3/css3-gradients.html)=> 
  - 线性渐变（Linear Gradients）- 向下/向上/向左/向右/对角方向
 - 径向渐变（Radial Gradients）- 由它们的中心定义
 - background: linear-gradient(to bottom right, red , blue); /* 标准的语法 */
- CSS3 文本效果
 - text-shadow
 - box-shadow
 - text-overflow CSS3文本溢出属性指定应向用户如何显示溢出内容
 - word-wrap 自动换行属性允许您强制文本换行 - 即使这意味着分裂它中间的一个字
 - word-break 单词拆分换行
- CSS3 字体 => [CSS3 @font-face 规则](http://www.runoob.com/css3/css3-fonts.html)
- CSS3 2D 转换
 - translate() translate()方法，根据左(X轴)和顶部(Y轴)位置给定的参数，从当前元素位置移动。
 - rotate() rotate()方法，在一个给定度数顺时针旋转的元素。负值是允许的，这样是元素逆时针旋转
 - scale() scale()方法，该元素增加或缩少的大小，取决于宽度（X轴）和高度（Y轴）的参数：
 - skew() 包含两个参数值skewX(< angle>), skewY(< angle>)，分别表示X轴和Y轴倾斜的角度，如果第二个参数为空，则默认为0，参数为负表示向相反方向倾斜
 - matrix() matrix 方法有六个参数，包含旋转，缩放，移动（平移）和倾斜功能。
- CSS3 3D 转换
 - perspective(n)	=> 定义 3D 转换元素的透视视图。
- CSS3 过渡
 - transition => 简写属性，用于在一个属性中设置四个过渡属性。 => 3
 - transition-property => 规定应用过渡的 CSS 属性的名称。 => 3
 - transition-duration => 定义过渡效果花费的时间。默认是 0。 => 3
 - transition-timing-function => 规定过渡效果的时间曲线。默认是 "ease"。 => 3
 - transition-delay => 规定过渡效果何时开始。默认是 0。 => 3
- CSS3 动画 => [CSS3 @keyframes 规则](http://www.runoob.com/css3/css3-animations.html)
- CSS3 多列
- CSS3 用户界面
 - resize => 属性指定一个元素是否应该由用户去调整大小
 - box-sizing => box-sizing 属性允许您以确切的方式定义适应某个区域的具体内容。
 - outline-offset => outline-offset 属性对轮廓进行偏移，并在超出边框边缘的位置绘制轮廓。

> 轮廓与边框有两点不同：
 - 轮廓不占用空间
 - 轮廓可能是非矩形
 [border、outline、boxshadow那些事以及如何做内凹圆](https://segmentfault.com/a/1190000005153660)
[CSS3如何实现圆角的outline效果？](http://www.zhangxinxu.com/wordpress/2015/04/css3-radius-outline/)


- CSS3 图片
	   - [响应式图片](http://www.runoob.com/try/try.php?filename=trycss_ex_images_responsive) => max-width: 100%;
	   - 图片滤镜 => [css滤镜](http://www.runoob.com/cssref/css3-pr-filter.html)
	   - [响应式图片相册](http://www.runoob.com/try/try.php?filename=trycss_image_gallery_responsive) 
	   - [图片 Modal(模态)](http://www.runoob.com/try/try.php?filename=trycss_image_modal_js)
- CSS3 按钮
	 -  [按钮动画](http://www.runoob.com/try/try.php?filename=trycss_buttons_animate1)
- CSS3 分页
- CSS3 框大小 =>  box-sizing 属性可以设置 width 和 height 属性中包含了 padding(内边距) 和 border(边框)。
	    - 关于 [border-box ](http://hexin.life/2017/07/11/title-15/) 
- CSS3 [弹性盒子](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)
- [CSS3 多媒体查询](https://developer.mozilla.org/zh-CN/docs/Web/Guide/CSS/Media_queries)
		  - viewport(视窗) 的宽度与高度
		  - 设备的宽度与高度
		  - 朝向 (智能手机横屏，竖屏) 。
		  - 分辨率 

## 垂直居中
### [大小不固定的图片、多行文字的水平垂直居中](http://www.zhangxinxu.com/wordpress/2009/08/%E5%A4%A7%E5%B0%8F%E4%B8%8D%E5%9B%BA%E5%AE%9A%E7%9A%84%E5%9B%BE%E7%89%87%E3%80%81%E5%A4%9A%E8%A1%8C%E6%96%87%E5%AD%97%E7%9A%84%E6%B0%B4%E5%B9%B3%E5%9E%82%E7%9B%B4%E5%B1%85%E4%B8%AD/)

## [圣杯布局](http://hexin.life/more/圣杯.html)

### 首先是 HTML 结构

```html
<div class="container">
        <div class="center">center</div>
        <div class="left">left</div>
        <div class="right">right</div>
    </div>
```
### CSS 布局
```css
     body,html {
            padding: 0;
            margin: 0;
        }

        .container {
            margin-right: 200px;
            margin-left: 200px;
        }
    
        .left,.center,.right{
            height: 300px;
            position: relative;
            float: left;
        }

        .left{
            width: 200px;
            right: 200px;
            margin-left: -100%;
            background-color: #ccc;
        }

        .center{
            width: 100%;
            background-color: red;
        }

        .right{
            width: 200px;
            margin-right: -100%;
            background-color: blue;
        }
```
## [双飞翼布局](http://hexin.life/more/双飞翼布局--三列中间自适应.html)
### 首先是 HTML 结构
```html
<div class="container">
 <div class="left">
        <p>左侧定宽</p>
    </div>
    <div class="main">
        <div class="center">
            <p>中间自适应</p>
        </div>
    </div>
    <div class="right">
        <p>右侧定宽</p>
    </div>
</div>
```
### CSS 布局
```css
 body,html{
        margin: 0;
        padding: 0;
        width: 100%;
    }
    .left {
        position: relative;
        width: 200px;
        height: 300px;
        float: left;
        margin-right: -200px;
        background-color: #ccc;
        text-align: center;
        line-height: 300px;
    }

    .right{
        position: relative;
        width: 200px;
        height: 300px;
        float: right;
        margin-left: -200px;
        background: red;
        text-align: center;
        line-height: 300px;
    }

    .main{
        width: 100%;
        float: left;
        background-color: blue;
        height: 300px;
        line-height: 300px;
        text-align: center;
    }

    .center{
        margin: 0 200px 0 200px;    
    }
```

### 更多
> [CSS布局中圣杯布局与双飞翼布局的实现思路差异在哪里？](https://www.zhihu.com/question/21504052)
> [双飞翼布局介绍-始于淘宝UED](http://www.imooc.com/wenda/detail/254035)
> [关于「圣杯布局」](https://segmentfault.com/a/1190000004524159)

## [Flex 实现圣杯布局](http://hexin.life/more/flex圣杯.html)
### 首先是 HTML 结构

```html
 <header>
        header
    </header>
    <div class="container">
        <div class="left">left</div>
        <div class="center">center</div>
        <div class="right">right</div>
    </div>
    <footer>
        footer
    </footer>
```

### CSS 布局

```css
body,html {
            padding: 0;
            margin: 0;
        }
        .all{
            display: flex;
            flex-direction: column; 
        }

        header,footer{
            flex: 1 0 auto;
            background-color: rgba(0,0,0,.4);
            height: 100px;
        }

        .container {
            display: flex;
            /* flex: 3;  */         
            /* 一共分为三份 */

            flex-direction: row;          
            /* 水平正向排列 */
        }
        
        /* 左右俩份，当屏幕缩小时，不缩小 */
        .left,.right{
            flex: 0 0 200px;           /*flex: x => {默认为0,有剩余空间不放大}  1 => {默认为为1，空间不足缩小}  size => {默认 auto ,占据固定空间}  => 后二可以省略*/
            background-color: blue;
            height: 200px;
        }

        .center{
            flex: 1;
            height: 200px;
            background-color: red;
        }
```
### 关于 flex 用法
> [Flex 布局语法教程](http://www.runoob.com/w3cnote/flex-grammar.html)
> [Flex 布局教程：语法篇--阮一峰](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html?utm_source=tuicool)
> [Flex 布局教程：实例篇](http://www.ruanyifeng.com/blog/2015/07/flex-examples.html)
# [CSS HTML 写作习惯/规范-网易NEC规范](http://nec.netease.com/)
# Node 

# 功能类

## [实现联动](http://hexin.life/more/linkage.html)
## [放大镜](http://hexin.life/more/mangnifyLens.html)

# 面试题整理

- [前端面试题及答案汇总](https://github.com/ivonzhang/Front-End-Developer-Questions/wiki)
- [前端开发面试题集锦](http://www.runoob.com/w3cnote/front-end-development.html)
- [2017年前端面试题最新汇总](http://blog.csdn.net/xllily_11/article/details/70899191)
- [30个你 “ 不可能全部会做 ” 的javascript题目及答案](http://developer.51cto.com/art/201504/474298.htm)
- [百度糯米前端技术学院](http://ife.baidu.com/college/detail/id/8)
- [newcode 牛客网](https://www.nowcoder.com/contestRoom)
- [lint-code](http://www.lintcode.com/zh-cn/problem/)
- [百度 FEX 的面试过程](https://github.com/fex-team/interview-questions)
- [Node interview of 饿了么](https://elemefe.github.io/node-interview/#/)
- [Front-end-Developer-Interview-Questions](https://github.com/h5bp/Front-end-Developer-Interview-Questions)
- [一些个面试题与js知识点](https://github.com/5Mi/wumi_blog/issues/69)
- [面试题](https://segmentfault.com/bookmark/1230000009049633)

> 知识整理及总结多来于网络及他人博客，由于时间原因，不能每一个知识点都完整的整理出来，因此附带了参考/学习链接。也等于一个知识点的梳理。
