---
title: 装饰器的使用
date: 2017/11/15
cover: https://cdn.jsdelivr.net/gh/jerryc127/CDN@latest/cover/default_bg.png
categories:
- Ecmascript
tags: 
- decorate
---
# 装饰器之助力开发

## 装饰器简介

+ Es7的装饰器是Object.defineProperty(obj, name, descriptor)的一个语法糖。
+ 装饰器提供定义劫持，能够对类及其方法、方法入参、属性的定义并没有提供任何附加元数据的功能。
+ 配合Reflect给了我们在类及其属性、方法、入参上存储读取数据的能力

> Decorators make it possible to annotate and modify classes and properties at design time.
>
> 官方给这个提案的评价是可以对类和类的属性进行注解和修改，这也是这一语法糖的价值所在



```javascript
function log(target, key, descriptor) {
    console.log(target);
    console.log(target.hasOwnProperty('constructor'));
    console.log(target.constructor);
    console.log(key);
    console.log(descriptor);
}

class Bar {
    @log;
    bar() {}
}

// {}
// true
// function Bar() { ...
// bar
// {"enumerable":false,"configurable":true,"writable":true}

```

关于方法装饰器的总结

+ target  -> className.prototType
+ Key -> method name
+ Descriptor  -> defineProperty
+ class decorate 情况下 target -> class 

defineProperty的基本内容如下：

- `configurable`控制是不是能删、能修改`descriptor`本身。
- `writable`控制是不是能修改值。
- `enumerable`控制是不是能枚举出属性。
- `value`控制对应的值，方法只是一个`value`是函数的属性。
- `get`和`set`控制访问咕噜的读和写逻辑。

### 利用这些简单的特性我们可以实现一个简单的单例

```javascript
function singleton (target) {
    return class Singleton extends target {
        constructor() {
          
        }
     static singleton = null；

    static getInstance(){
             if(singleton == null) {                        
                 singleton = new Singleton ();  
             }
             return singleton ；

     }
    }
}

@singleton
class DoA {
    // ....
    runSql(){}
}

DoA.getInstance().runSql();

@singleton
class DoB {
    // ....
    runSql(){}
}

DoB.getInstance().runSql();
```





### 与spring的annotation比较

在Java中有一种提供元信息的方式，叫做注解，如果用spring写一个route：

```java
@RestController
@SpringBootApplication
public class BookApplication {

  @RequestMapping(value = "/available")
  public String available() {
    return "Spring in Action";
  }

  @RequestMapping(value = "/checked-out")
  public String checkedOut() {
    return "Spring Boot in Action";
  }

  public static void main(String[] args) {
    SpringApplication.run(BookApplication.class, args);
  }
}
```



这种命令式的模式非常的直观，最后，用装饰器的方式实现一个基于express的route。

```javascript
// collectRoute.js
// 用容器存储Route的信息
const routerList = [];
const logger = require('../log/logger');


function Route(params) {
  return function (target) {
    try {
      const methodObj = target.prototype;
      if(routerList.filter(v => v.prefix === params) !== []) {
        routerList.push(Object.assign({ prefix: params }, methodObj));
      } else throw new Error('the same route name is not allow');
    } catch (e) {
      logger.error(e);
    }
  };
}

module.exports = {
  Route,
  routerList,
};


// httpMethod.js
// 用method 装饰器存储http method 的描述信息
const logger = require('../log/logger');

const method = ['GET', 'POST', 'PUT', 'DELETE'];
let methodList = [];
const totalMethod = {};
method.forEach((v) => {
  totalMethod[v] = function (params) {
    methodList = [];
    return function (target, name, descriptor) {
      try {
        const old = descriptor.value;
        methodList.push({ method: v.toLowerCase(), path: params, handle: old });
        target.methodList = methodList;
        descriptor.value = function () {
          return old(...arguments);
        };
        return descriptor;
      } catch (e) {
        logger.error(e);
      }
    };
  };
});

module.exports = totalMethod;

// server.js
// 结合express遍历记录的信息
routerList.forEach((routerItem) => {
  const router = require('express').Router();
  const { methodList } = routerItem;

  methodList.forEach((methodDescription) => {
    const { method, path, handle } = methodDescription;
    router[method](path, (req, res) => {
      handle(req, res);
    });
  });

  app.use(routerItem.prefix, router);
});

//Controller/User.js

// 实现命令式的Http method
const { Route } = require('../core/collection-router');
const { GET, PUT } = require('../core/http-method');

@Route('/user')
class User {
  @GET('/:id')
  findWordBy(req, res) {
    const { params } = req;
    res.send(params.id);
  }

  @PUT('/:id')
  updateWordBy(req, res) {
    const { params } = req;
    res.send(params.id);
  }
}

module.exports = {
  UserRouter: User
};

```

