title: 混合继承
date: 2017/01/13
categories:
- js
tags:
- js

---
# 混合继承

```javascript
/*  var obj =new Object();      //用于创建对象的

  console.log(obj.constructor );  //检测实例的构造函数是哪个

                  //实例会将对象中所有的属性都拷贝一份

  function obj1 () {

    this.name = "mark"

  }

  obj.constructor = obj1.constructor;

  console.log(obj.constructor);//是构造函数的一个隐藏属性*/

  //对象是function的一个实例

  //原型属性和方法被所有实例共享

  //被创造出来就具备prototype 和 constructor 和function对象中的一切属性和方法

  //实例化的时候会拷贝构造函数中的方法

  //prototype中保存的是地址 将实力和原型联系在一起

  //指向相关就是指向地址

  function man () {

    this.dick = "big"

  }

  man.prototype = {

    funk : function () {

      console.log('每秒一百下');

    }

  }

  var black = new man();

  var white = new man();

//当调用一个方法时首先在自己的prototype中查找 ，找不到时沿着construct 中的 proto向父类的原型中去寻找

//prototype 包含一个对象的构造方式和自己的方法

//delete 暴露原型

  console.log(black.proto === man.prototype);

</script

正则 注册  formatString的实现

/*var html = template("jsId{{}}","json")

document.getElementById("htmlId").innerHtml = html*/

模板

//简易模板

var data = [

  {

    "name" : :"mark",

    "age" : 25},

  {

    "name" : :"mark",

    "age" : 25},

  {

    "name" : :"mark",

    "age" : 25}]

  var source = "<h1>{{name}}</h1>"+

         "<h1>{{age}}</h1>"

  var render = template.compile(source);

  var html = render(data);

  function Temp (source,data) {

    var render = template.compile(source);

    var html = render(data);

    return html

  }

-----------------------------------------------

var a = function () {

  this.say = function () {

    alert(1);

  }

}

var bb = new a();

var cc = {

  "say" : function(){

    alert(1);

  }

}

cc.proto = a.prototype

bb.say();//1

cc.say();//1

```



***!!!!构造函数不通过__proto__ 继承***

***！！new 实例一个函数  1新建对象  2让新建对象的__PROTO__ 指向 prototype（原型只继承原型内的 ， 不继承构造函数中的内容） 3继承方法***

***1只有函数对象拥有原型属性***

***2Foo.prototype = {}会创建一个对象导致 原型内的隐式属性发生变化改变构造器的内容  由Foo 变至 Object***                      

![image-20210326174456631](https://technologybook.tech/assets/img/image-20210326174456631.png)

\#构造函数


![image-20210326174535252](https://technologybook.tech/assets/img/image-20210326174535252.png)