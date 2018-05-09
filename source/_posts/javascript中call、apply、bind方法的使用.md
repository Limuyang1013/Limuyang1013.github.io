---

title: javascript中call、apply、bind方法的使用

date: 2017-11-08 20:50:19

categories: 前端开发

tags: [javascript,call,apply,bind]


toc: true

---

![](https://www.tuchuang001.com/images/2018/04/27/maxresdefault.jpg)

<!--more-->
#### context的概念
在知道我们为什么要使用call、apply、bind方法之前，我觉得有必要先了解一下context的相关概念，通常context的作用是取决于函数将如何被调用，当函数作为对象的方法调用时，this就会被设置为调用方法的对象：
```javascript
 var object = {
        foo: function(){
            console.log(this === object);
        }
    };

    object.foo(); // true
```
当通过new一个对象的实例的方式来调用一个函数的时候，this的值将被设置为新创建的实例：
```javascript
  function foo(){
        console.log(this);
    }

    foo() // window
    new foo() // foo{}
```
当作为未绑定对象被调用时，this默认指向全局上下文或者浏览器中的window对象。然而，如果函数在严格模式下被执行，上下文将被默认为undefined

#### 动态改变this的值

**然而call、apply、bind方法允许你在自定义的context中执行函数**，什么意思呢，通俗的说就是可以不遵循上面给出的部分定义，可以动态的改变this，我们直接引出MDN关于call方法的定义：
> call() 方法调用一个函数, 其具有一个指定的this值和分别地提供的参数(参数的列表)

举个例子：
```javascript
// 非严格模式下
function add(args) {
        console.log(this);
    }
    function sub(args) {
        console.log(this);
    }
    add(); // Window {frames: Window, postMessage: ƒ, blur: ƒ, focus: ƒ, close: ƒ, …}
    sub(); // Window {frames: Window, postMessage: ƒ, blur: ƒ, focus: ƒ, close: ƒ, …}
```
结果是符合预期的：
> 当作为未绑定对象被调用时，this默认指向全局上下文或者浏览器中的window对象

我们给他加上call方法：
```javascript
// 非严格模式下
 function add(args) {
        console.log(this);
    }
    function sub(args) {
        console.log(this);
    }
    add(); // Window {frames: Window, postMessage: ƒ, blur: ƒ, focus: ƒ, close: ƒ, …}
    sub(); // Window {frames: Window, postMessage: ƒ, blur: ƒ, focus: ƒ, close: ƒ, …}
    add.call(sub,1); // ƒ sub(args) { ... }
    sub.call(add,1); // ƒ add(args) { ... }
```
这时候this的值有点类似于第一种情况：
> 当函数作为对象的方法调用时，this就会被设置为调用方法的对象

看起来我们调用call方法的时候好像实际上是进行了对象方法的调用：

```javascript
    sub.call(add,1);  //sub
    =>
    var sub = {
        add: function(){
            console.log(this);
        }
    };
    sub.add(); // sub
```
这就是动态的改变了this
还有一点要注意的是，这里call和apply方法第一个参数的this值并不一定是该函数执行时真正的this值，比如在非严格模式下，我们传入`null`或`undefined`会自动指向全局对象，在浏览器环境中就是`window`，在node环境中就是`global`，另外我们还可以传入一些原始值(数字，字符串，布尔值)，这时候的this会指向这些原始值的包装对象，比如传入的是数字，会指向Number。

#### call和apply的区别

既然call和apply方法能够动态的改变this的值，我们可以利用这个特性来实现简单的继承，比如此时我们有一个父构造函数：
```javascript
function Product(name, age) {
        this.name = name;
        this.age = age;
        this.say = function() {
            console.log(this.name+"is"+this.age+"years old");
        }
    }
```
如果我们想写一个子构造函数，只需要在子构造函数里面调用父构造函数的call方法就可以实现继承：
```javascript
function Toy(name, price) {
        Product.call(this, name, price);
        this.self = "single";
    }
```
实现继承的同时父子构造函数还都能分别有自己的属性，比如这里的self属性。
这里使用了call方法作为演示，其实apply方法的使用和call方法并无多大区别，重点在参数上，他们的第一个参数都是一个指定的this值，这个值可以是任意js对象，但是第二个参数有区别，call方法接受的是若干个参数的列表，而apply方法接受的是一个包含多个参数的数组，如果上面的例子用apply来写的话就是这样：
```javascript
function Product(name, age) {
        this.name = name;
        this.age = age;
        this.say = function() {
            console.log(this.name+"is"+this.age+"years old");
        }
    }

    function Toy(name, price) {
        Product.apply(this, [name, price]);
        this.self = "single";
    }
```

#### bind方法的特殊性
之所以把bind方法单独放出来是因为bind方法和前面两者还是有不小的区别的，虽然都是动态改变this的值，举个例子：
```javascript
   var obj = {
        x: 81,
    };

    var foo = {
        getX: function() {
            return this.x;
        }
    }
    console.log(foo.getX.bind(obj)());  //81
    console.log(foo.getX.call(obj));    //81
    console.log(foo.getX.apply(obj));   //81
```
有没有注意到使用bind方法时候后面还要多加上一对括号，因为使用bind只是返回了对应函数并没有立即执行，而call和apply方法是立即执行的，并且MDN还提到，**当使用new操作符调用绑定函数时指定的this参数会变无效：**
```javascript
 var obj = {
        x: 81,
    };

    var foo = {
        getX: function() {
            return this.x;
        }
    }
    var result = foo.getX.bind(obj);
    console.log(new result());  //getX
```
bind方法的另一个应用是使一个函数拥有预设的初始函数，这些参数作为bind的第二个参数跟在this(或其他对象)后面，之后它们会被插入到目标函数的参数列表的开始位置，传递给绑定函数的参数会跟在它们的后面：
```javascript
function list() {
  return Array.prototype.slice.call(arguments);
}

var list1 = list(1, 2, 3); // [1, 2, 3]

// Create a function with a preset leading argument
var leadingThirtysevenList = list.bind(undefined, 37);

var list2 = leadingThirtysevenList(); // [37]
var list3 = leadingThirtysevenList(1, 2, 3); // [37, 1, 2, 3]
```
#### 参考链接
[Understanding Scope and Context in JavaScript](http://ryanmorr.com/understanding-scope-and-context-in-javascript/)

[Function.prototype.call()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/call)
