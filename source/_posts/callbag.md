---
title: callbag二三事
date: 2020/01/15
cover: https://loreanvictor.github.io/callbag-jsx/docs/assets/callbag-jsx.svg
categories:
- Reactive
  tags:
- callbug
---
# callbag 源码分析

> Your focus determines your reality.

callbag组合了一些基础的***设计模式***，例如观察者模式和迭代器模式等，通过这些设计模式来达到管理序列任务的目的。

## 观察者模式

```typescript
	interface Publisher {
    subscribe(suberscriber Suberscriber): void;
    unSubscribe(suberscriber Suberscriber): void;
    notify(): void;
  }

  interface Suberscriber {
    update(data: any): void
  }

  class concretePublisher implement Publisher {
    private subscribers: Suberscriber[] = []；
    
    subscribe(suberscriber Suberscriber): void {
      !this.suberscribers.includes(suberscriber) && this.subscribers.push(suberscriber)
    };

  	unSubscribe(suberscriber Suberscriber): void {
    	this.suberscribers.includes(suberscriber) && this.subscribers.splice(suberscriber, 1)
  	};

  	notify(): void {
    	this.suberscribers.forEach(f => f.update())
    };
  }

class concreteSuberscriber implement  Subscriber {
  update(...data){
    console.log(data)
  }
}
```

> 优点 
>
> - *开闭原则*。 你无需修改发布者代码就能引入新的订阅者类 （如果是发布者接口则可轻松引入发布者类）。
> - 可以在运行时建立对象之间的联系
>
> 缺点
>
> - 时序随机
>
> 

## 迭代器模式

```typescript
interface Iterator {
  isDone(): boolean
  next(): {value: any, done: boolean}
}

class Iterator implement Iterator{
  private index = 0;
  constructor(private value: any[]) {
    this.value = value;
  }
  isDone() {
    return this.index >= this.value.length 
  }
  next() {
    if (!this.isDone()) {
      this.index++
      return {
        value: this.value[this.index],
        done: false
      }
    }
    return 
        value: this.value[this.index],
        done: true
      }
  }
}
```

### 推拉模型

推（pushable）： 即**Publisher是主动方**，而**Subscriber是被动方**，subscriber并不知道什么时候什么时间接收数据。

拉（pullable）： 需要由Sub儿scriber决定publisher不知道何时接受数据。

##  基本原理

### sink

```typescript
// sink 接受两个params, type 和 payload

// type的三种类型：
// 0 开始
// 1 数据传输阶段
// 2 结束

payload，可以传递 具体 data 或者 sink

type sink = (type: type, payload: any})： void
enum type {
  start = 0,
  data,
  end
}
```



***sink 建立连接前，必须先与进行一次 type 为  0  handleshake， 类似tcp握手然后 （type = 1）开始传递相应的payload,当任意一边传递了2（结束信号）则结束。***

### TODO: myCallbag 简单实现







