title: This 
date: 2017/01/12
categories:
- js
tags:
- js

---
# Javascript this

# 一.Global this



1，在浏览器里，**全局情况下this等同于window对象。**
2，当没有var 或 let 时 对变量的修改就是在改变this 的属性值。
3，node环境里 一般的会给global this 同时赋值。
4，node环境中 逐行读取的情况中 会优先给Global赋值。

# 二.Function this

1，如果调用函数时使用new 那么它就会成为一个新值。
2，如果**正常使用this 那么他就指代一个全局的this** 如果**添加一个new** 那么它就成为了**构造函数**并创建了一个实例，this指代这个实例（**如果你在调用函数的时候在前面使用了new，this就会变成一个新的值，和global的this脱离干系**）。
例如( setTimeout, setInterval,addEventListenner),通常使用self 来解决这个问题。

# 三.Prototype this

1，**每创建一个函数对象**就会**自动获得一个特殊属性prototype** 通过**new 调用一个函数就能通过this 访问这个值**。

```
 function Thing() {
        console.log(this.foo);
         }
 Thing.prototype.foo = "bar";
 var thing = new Thing(); //logs "bar"
 console.log(thing.foo);  //logs "bar"（原型链）
```

2，可以通过在实例里面删除自己添加的属性的方式来实现
Thing.prototype.deleteFoo = function () {
delete this.foo;
}
3，可以把多个函数 的prototype 链接起来从而形成一个原型链 这样this会沿着这条原型链网上查找知道找到你想要的值。（原型继承）
4，在prototype里面定义的方法会**影响到当前实例上游的this** 这意味着**当你赋值时this时隐藏了上游的属性**（Ting1的实例中的**proto**指向Thing2的prototype属性建立原型链）

```
function Thing1() {
 }
Thing1.prototype.foo = "bar";
Thing1.prototype.logFoo = function () {
    console.log(this.foo);
     }
function Thing2() {
 this.foo = "foo";
     }
*   Thing2.prototype = new Thing1();
```

5，**嵌套函数可以捕获父元素的变量，但时这个函数没有继承this（通常使用 var _this = this来结局这个问题）**
6，**通过bind 可以将实例与方法一同传给函数**

```
 function Thing() {
                     }
 Thing.prototype.foo = "bar";
 Thing.prototype.logFoo = function () { 
         console.log(this.foo);
         }
 function doIt(method) {
        method();
         }       
var thing = new Thing();
//console.log(aStr , this.foo);
 doIt(thing.logFoo.bind(thing)); //logs bar
logFoo.bind(thing)("using bind"); //logs "using bind bar"
logFoo.apply(thing, ["using apply"]); //logs "using apply bar"
logFoo.call(thing, "using call"); //logs "using call bar"
logFoo("using nothing"); //logs "using nothing undefined"
```

7,可**以通过var thing = Object.create（Thing.prototype）来创建一个实例**，这种创建方式**在继承模式下重写构造函数**非常有用。Thing2.prototype = Object.create(Thing1.prototype);（通过new操纵继承 ，通过object.create继承**不会**调用构造函数）

# 四.Object this

1，只 有有相同的**直接父元素**的属性才能通过this共享变量（两次黏连就近原则）
2，可以直接通过对象引用你需要的属性

```
var obj = {
              foo: "bar",
              deeper: {
              logFoo: function () {
              console.log(obj.foo);
             }
          }
        };
    obj.deeper.logFoo(); //logs "bar"
```

四.DOM event this
1，在一个HTML DOM事件处理程序里边 ， **this 始终指向这个处理程序被所绑定到的HTML DOM 节点** （除非使用了bind）

# 五.HTML & override & jQuery & thisArg this

1，this 是保留字不能够重写
2，eval可以访问this
3，HTML 中可以放置代码并指向这个元素
4，你可以通过with来将this添加到当前的执行环境，并且读写this的属性的时候不需要通过this
5，**jQuery 中this都是指向HTML 元素节点 及应用**

```
 1 <div class="foo bar1"></div>
 2 <div class="foo bar2"></div>
 3 <script type="text/javascript">
 4 $(".foo").each(function () {
 5     console.log(this); //logs <div class="foo...
 6 });
 7 $(".foo").on("click", function () {
 8     console.log(this); //logs <div class="foo...
 9 });
10 $(".foo").each(function () {
11     this.click();
12 });
13 </script>  
```

thisArgs bind call apply thisArgs传参给函数 这个this 绑定 为你传递的对象。
