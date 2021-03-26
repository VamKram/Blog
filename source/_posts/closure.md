title: closure && object.create
date: 2017/01/04
categories:
- js
tags:
- js

---


# closure && object.create

### 闭包就是能够读取其他函数内部变量的函数

**f2可以使用f1中的参数当f1的返回值为f2时 外部函数可以调用内部的参数这就形成了一个闭包。**

```javascript
function f1(){
　　　　var n=999;
　　　　function f2(){
　　　　　　alert(n); 
　　　　}
　　　　return f2;
　　}
　　var result=f1();
　　result(); // 999
```

### 闭包的用处

> 可以读取函数内部的变量，
> 让这些变量的值始终保持在内存中。(通常情况下应用完销毁)

```javascript
function f1 () {
        var n = 5;
        add = function(){n+=1}
         function f3(){
            alert(n);
        }
        return f3;
    }
    var f2 = f1();
    f2();  // 5
    add(); // +1
    f2(); // 6 内存中的n没有被销毁
```

### 使用闭包的注意点

> 由于闭包会使得函数中的**变量都被保存在内存中**，**内存消耗很大**，所以不能滥用闭包，否则会造成网页的性能问题，在IE中可能导致内存泄露。解决方法是，**在退出函数之前，将不使用的局部变量全部删除**。
> *(不清除内存)*
> 闭包会在父函数外部，**改变父函数内部变量的值**。所以，如果你把父函数当作对象（object）使用，把闭包当作它的公用方法（Public Method），把内部变量当作它的私有属性（private value），这时一定要小心，不要随便改变父函数内部变量的值。

```javascript
　　var name = "The Window";
　　var object = {
　　　　name : "My Object",
　　　　getNameFunc : function(){
　　　　　　return function(){
　　　　　　　　return this.name;
　　　　　　};
　　　　}
　　};
　　alert(object.getNameFunc()());
　　————————————————
　　var name = "The Window";
　　var object = {
　　　　name : "My Object",
　　　　getNameFunc : function(){
　　　　　　var that = this;
　　　　　　return function(){
　　　　　　　　return that.name;
　　　　　　};
　　　　}
　　};
　　alert(object.getNameFunc()());
```

## object.create

1，定值为null 创建不带原型对象
2，创建带属性带原型的对象

```javascript
var pt = {
        say : function(){
            console.log('saying!');    
        }
    }

var o3 = Object.create(pt, {
        size: {
            value: "large",
            enumerable: true
        },
        shape: {
            value: "round",
            enumerable: true
        }    
    });

    console.log(o3.size);    // large
    console.log(o3.shape);     // round
    console.log(Object.getPrototypeOf(o3));     // {say:...}
```

3，实现继承

```javascript
//Shape - superclass
        function Shape() {
          this.x = 0;
          this.y = 0;
        }

        Shape.prototype.move = function(x, y) {
            this.x += x;
            this.y += y;
            console.info("Shape moved.");
        };

        // Rectangle - subclass
        function Rectangle() {
          Shape.call(this); //call super constructor.继承属性
        }

        Rectangle.prototype = Object.create(Shape.prototype);//继承方法

        var rect = new Rectangle();

        console.log(rect instanceof Rectangle); //true.
        console.log(rect instanceof Shape); //true.

        rect.move(); //"Shape moved."
        //new object（）没有父类 使用其构造方法和属性
        //Object.create（）指向原型 （父子关系）
```

## call apply bind

1，将参数传递给apple

```javascript
banana = {
    color: "yellow"
}
apple.say.call(banana);     //My color is yellow
apple.say.apply(banana);    //My color is yellow
```

2，

> 某个函数的参数数量是不固定的，因此要说适用条件的话，当你的参数是明确知道数量时用 call 。
> 不确定的时候用 apply，然后把参数 push 进数组传递进去。当参数数量不确定时，函数内部也可以通过 arguments 这个数组来遍历所有的参数。
> 如果第一个参数为null，函数体内的this指向宿主对象，在浏览器中是window

结构 fn.prototype.apply(this,arguments)
*arguments是this中的参数 再传入fn中。fn.prototype.apply(this—>argument)*
3，**bind**
**bind()方法会创建一个新函数**，称为**绑定函数**，当调用这个绑定函数时，绑定函数会以创建它时传入 **bind()方法的第一个参数作为 this**，**传入 bind() 方法的第二个以及以后的参数加上绑定函数运行时本身的参数按照顺序作为原函数的参数来调用原函数。**
*多次 bind() 是无效的。*

> apply 、 call 、bind 三者都是用来改变函数的this对象的指向的；
> apply 、 call 、bind 三者第一个参数都是this要指向的对象，也就是想指定的上下文；
> apply 、 call 、bind 三者都可以利用后续参数传参；
> bind 是返回对应函数，便于稍后调用；apply 、call 则是立即调用 。