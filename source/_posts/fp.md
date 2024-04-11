title: JavaScript设计模式
date: 2017/01/12
categories:
- js
tags:
- js

---
# JavaScript设计模式

## 工厂模式

1，**用函数来封装特定接口创建对象的细节**。工厂模式解决了**创建多个相似对象对象的问题**，但还没有解决**对象识别的问题**。（通过函数封装）

```
function createPerson(name, age, job) {
    var o = new Object();//用以封装接口的函数
    o.name = name;
    o.age = age;
    o.job = job;
    sayName = function () {
        alert(this.name);
    };
    return o;
}
var person1 = createPerson('zxj', 23, "Software Engineer");
var person2 = createPerson('sdf', 25, "Software Engineer");
```

## 构造函数模式

1，Person()中的代码除了和createPerson()中相同的部分外，还存在以下不同之处：

- 没有显式的创建对象；
- 直接将属性和方法赋给了this对象；
- 没有return语句；
  2，**创建构造函数Person的实例必须使用new运算符**：
- 创建一个新对象；
- **将构造函数的作用域赋给新对象**（因此this就指向这个新对象）；
- 执行构造函数中的代码（为这个新对象添加属性和方法）；
- 返回新对象；

```
function Person () {
    this.name = name;
    this.age = age;
    this.job = job;
    this.sayName = function(){
           alert(this.name); 
   }
}
    var person1 = new Person("mark" , 29 , 'Software Engineer');
    var person2 = new Person("Greg" , 27 , 'Doctor');
```

3,**实例化的person1 ，person2 都有一个constructor 属性指向Person**(标示对象类型用constructor 检测对象类型用instanceof)

> alert(person1.constructor == Person ); //true
> alert(person1.constructor == Person ); //true
> 通过构造的constructor 指向Person，通过原型使用**proto** 指向原型的prototype。

4，构造函数与函数
1， 两者的调用方式不同，构造函数**只能通过new操作符来调用**

```
// 当作构造函数
var person = new Person('zxj', 23, "Software Engineer");
person.sayName(); //zxj 
// 当作普通函数
Person('sdf', 25, "Software Engineer"); //添加到window
window.sayName(); //saf
// 在另一个对象的作用域中调用
var o = new Object();
Person.call(o, "qwe", 25, "Nurse");
o.sayName(); //qwe
```

2，构造函数的问题

```
function Person(name, age, job) {
    this.name = name;
    this.age = age;
    this.job = job;
    this.sayName = sayName;//被调用的函数
}
//相当于定义了全局的函数在构造函数体内被调用
function sayName() {
    alert(this.name);
};
//每个Person 实例都包含一个不同的FUNCTION以这种方式创建函数会导致不同的作用域链和标识符解析
var person1 = new Person('zxj', 23, "Software Engineer");
var person2 = new Person('sdf', 25, "Software Engineer");
```

# 原型模式

1，每个函数都有一个prototype属性这个属性**包含一个\*_proto_\***这个属性是**一个指针**，**指向一个父对象的prototype**属性，而这个对象可以和特定实例共享属性和方法。

```
function Person(){
}
Person.prototype.name = 'Nicholas';
Persaon.prototype.age = 29;
Person.prototype.sayName = function (){
    alert（this.name）;
}
var person1 = new Person();
person1.sayName();//"Nicholas";
var person2 = new Person();
person2.sayName();//"Nicholas"
alert(person1.sayName == person2.sayName) // true
```

2，**只要创建了一个新函数**就会有一组特定的规则为该函数**创建一个prototype**属性 这个属性中的**constructor指向f父对象描述了这个函数的构造方法**，***_proto_\*指针指向父对象的prototype**
*可以通过isPrototypeof方法来确定对象之间是否存在这种关系该方法返回一个Boolean值*

> alert(person.prototype.isPrototypeof(person1))
> alert(person.prototype.isPrototypeof(person2))

3，原型模式实现原理 每当代码读取某个属性时，都会执行一次搜索，目标是具有给定名字的属性。搜索首先从**实例**本身开始。如果在**实例**中找到了具有给定名字的属性。如果查找到则返回，否则则在父对象的prototype中寻找通过 **proto**属性。（实例）

4，delete可以删除来自原型的的值然后显示出继承自父对象的属性。

```
var person1 = new Person();
var person2 = new Person();
person1.name  = "mark"
alert(person1.name) //"mark"
alert(person2.name) // "Nichloas"
delete person1.name ; 
alert(person1) // "Nichloas"
```

5 ,**hasOwnProperty()**是检测一个属性是存在于实例中还是存在与原型中（这个方法继承自Object对象） 该方法返回一个Boolean值

```
function Person () {
}
Person.prototype.name = "Nichloas"
Person.prototype.age = "Software Engineer ";
Person.prototype.sayName = function(){
        alert(this.name)
}
var person1 = new Person();
var  person2 = new Person();
person1.name  = "Gerg"
alert (person.hasOwnProperty("name")); //false 属性来自实例
alert(person2.name)
alert(person2.hasOwnProperty("name"))://true 属性来自原型
delete person1.name
alert(person1.hasOwnProperty("name"))//true 属性0000000来自原型
```

### 原型与in操作符

> 在for in 循环中使用
> 单独使用时in 操作符通过对象能够访问给定属性时返回true 与hasOwnProperty不同in只判断是否访问到而后者判断是否来自原型

```
*判断存在于对象中或者存在于原型中代码*
function hasProtootypeProperty(){
     return !object .hasOwnProperty (name ) && (name in object)
}
```

#### 在使用枚举 for in 时会返回可枚举的（enumerated）所有属性

*ECMAscript5 将constuctor 和 prototype 属性[Enumerable]属性设置成false 不可枚举*
要取得对象上所有可枚举的实例属性 可以使用Object.key()方法

```
var key = Object.key(Person.prototype);
```

要取得所有实例属性（无论可否枚举）使用Object.getOwnPropertyNames()方法

```
 var key = object.getOwnPropertyNames(Person.prototype);
```

(都可以替代for in 来获取实例属性)

# 更简洁的的与原型语法

Person.prototype设置成为等于一个对象字面量形式创建的对象，相当于重构了Person.prototype。得到的结果相同。但constructor不再指向Person而是指向最初的Person原型
*重写了prototype语法*
先创建实例后修改原型依旧运转正常（**原型中查找值的过程是一次搜索**）

```
var friend = new Person();
PersonPrototype.sayHi = function() {
        alert("hi")
}
friend.sayHi();
```

**原型的缺点**

- 实例在默认情况下取得相同的属性值，所有属性被实例共享，对于包含引用类型的属性来说是一场灾难。

```
 function Person () {
 }
 Person.prototyrpe={
//重写prototype属性
friends:["mark","lee"],
}
var person1 =new Person();
var person2 =new Person();
person1.friends.push（“van”）
alert(person2.friends)//mark lee van
```

##### ！！！prototype 的指向确定属性，constructor确定构造方法