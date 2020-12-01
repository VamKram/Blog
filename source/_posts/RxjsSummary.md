---
title: Rxjs 总结
date: 2017/09/09
categories:
- reactive programming
tags: 
- rxjs

---
# Rxjs 总结（一）
![rx](https://cdn-images-1.medium.com/max/2000/1*gD37OB2-PtMqZdk3X1YnEQ.png)

# 操作符汇总

> 在Rx的世界发生的一切现象都是围绕着数据流展开的，数据流以Observable对象的的形式存在，而对Rx的应用实际就是使用操作符对数据流进行操作。按照使用方式，大体上可以将操作符划分为四类，分别是创建类，合并类，过滤类，转化类操作符。

## 创建类操作符

> 顾名思义，创建类数据流的产出是一个Observable对象。

#### 一. 创建同步数据流

**create** :create的本质是直接调用Observable的构造函数。

```javascript
Observable.create = function(subscribe) {
    return new Observable(subscribe)
}
```

 **of**  : 利用of操作符可以轻松产生包含给定数据集合的Observable对象。

```javascript
Rx.Observable.of(1,2,3).subscribe(console.log);
//output: 1,2,3
```

**range** : 返回一定范围上，以第一个参数为起点，持续加一作为数据的Observable对象

```javascript
Rx.Observable.range(1,100).subscribe(console.log);
//output: 1......100
Rx.Observable.range(1.5,100).subscribe(console.log);
//output: 1.5,2.5,3.5....100.5
```

**generate**  : 类似于for循环，包含条件判断，递增值设置，返回结果规则，创建一组Observable。

```javascript
Rx.Observable.generate(2,v <= 10, v => v+2, v => v**2).subscribe(console.log)
// output 4, 16, 36, 64, 100
```

**repeat** ： repeat操作符是一个实例操作符，能够复制上游的Observable中的数据若干次，使用repeat操作符实际上是将Observable订阅了若干次	。

```javascript
Rx.Observable.of(1,2,3).repeat(3).subscribe(console.log);
// output 1,2,3,1,2,3,1,2,3
```

**empty** : 产生一个空的直接完结的Observable对象。

**throw** ： 产生一个直接抛出错误的Observable对象。

```javascript
Rx.Observable.throw(new Error('Test')).subscribe(console.log, console.log, console.log('complete'))

// output complete 
// Error: Test
```

**never** : 产生一个既不抛错也不完结，也没有什么动作，就那么呆着的一个Observable。。。。。

```javascript
Rx.Observable.generate(2,v => v <= 10, v => v+2, v => v**2).subscribe(console.log)
// output 4, 16, 36, 64, 100
```

#### 二. 创建异步数据流

**interval** ： 创建出一个根据设置时间递增的Observable对象，interval不会主动停止，需要下游调用complete才会终止。

```javascript
Rx.Observable.interval(1000).subscribe(console.log)
// output 1，2，3.....
```

**timer** ： （时间/毫秒数, ？时间间隔）=> 若没有第二个参数，timer会在指定时间返回0，若有第二个参数则以第一个参数为起点，第二个参数为时间间隔，持续产生一个递增的Observable对象。

```javascript
Rx.Observable.timer(Date.now() + 1000).subscribe(console.log)
// output 0
Rx.Observable.timer(Date.now() + 1000, 1000).subscribe(console.log)
// output 0, 1, 2, 3, 4
```

#### 三. 把其他类型的数据转化为Observable

**from** : 可以把任何js对象转化为Observable对象。

```javascript
Rx.Observable.from([1,2,3]).subscribe(console.log);
// output: 1, 2, 3
```

**fromPromise** : 将promise类型的对象转化为Observable 对象；

```javascript
Rx.Observable.fromPromise(Promise.resolve("test")).map(v => `hello, ${v}`).subscribe(console.log)

// output: hello test
```

**fromEvent** : 将DOM事件，或nodejs的event对象转化为Observable；

```javascript
Rx.Observable.fromEvent(document.getElementById('app'), 'click').subscribe(console.log);
// node 
const emitter = new EventEmitter();
Rx.Observable.fromEvent(emitter, 'test').subscribe(console.log);
emitter.emit('msg', 1);
// output: 1

```



#### 四. 其他

**repeatWhen** ： 相比于repeat可控的重复订阅上游。

```javascript
Rx.Observable.of(1,2,3).repeatWhen((notifier$) => notifier$.delay(1000)).subscribe(console.log)
// output: 1,2,3  1,2,3
```

**defer** : 延迟Observable创建，仅在被订阅的时候创建。

```javascript
const fetchData = () => Observable.ajax(url);
const source$ = Rx.Observable.defer(fetchData);
// 仅当source被订阅时才发出ajax请求
```

## 合并类操作符

> 随着需求的复杂化，单一的操作符很难满足需求，因此操作符之间的组合让Rx发挥更大的威力。

| 功能                                     | 操作符                                     |
| ---------------------------------------- | :----------------------------------------- |
| 把多个数据流以先到先得的方式合并         | merge， mergeAll                           |
| 把多个数据流首尾相接的方式合并           | concat，concatAll                          |
| 把多个数据流以一一对应的方式合并         | zip，zipAll                                |
| 持续合并多个数据流中最新产生的数据       | combineLatest， combineAll, withLatestFrom |
| 从多个数据流中选取第一个产生内容的数据流 | race                                       |
| 在数据流前面添加一个指定数据             | startWith                                  |
| 只获取多个数据流最后产生的那个数据       | forkJoin                                   |
| 从高阶数据流中切换数据源                 | switch， exhaust                           |

**concat** ： 首尾相接

```javascript
Rx.Observable.of(1,2,3).concat(Observable.of(4,5,6)).subscribe(console.log)
// output: 1,2,3,4,5,6
// 如果Observable对象没有complete则永远不会concat到下一个Observable
```

**merge** ： 先到先得

```javascript
const mouseUp$ = fromEvent(document, 'mouseup');
const mouseDown$ = fromEvent(document, 'modedown');
const source$ = merge(mouse$, mouseDown$);
// 鼠标点击则触发订阅的发生
/*
*@params Observable
*@params Observable
*@params number {可容纳Observable的数量}
*/
// 常用作合并异步数据流，http request， dom event等。
```

**zip** ： 一一对应

```javascript
//zip 会将上游数据转化为数组形式
// 只要zip上游任何一个Observable完结， zip就会完结
Observable.zip(of(1,2,3), of('a', 'b', 'c')).subscribe(console.log);
// output [1,a], [2,b], [3,c]

```

**combineLatest** ： 合并最后一个数据

```javascript
const run1$ = timer(500, 1000);
const run2$ = timer(1000, 1000);
combineLatest(run1$, run2$).subscribe(console.log);
// output: [0,1], [1,1]...
//每隔500毫秒产出一次数据， 最新最近有数据流流过则触发combineLatest的subscribe
```

**withLatestForm** : 仅根据上游的单一数据源向下游推送数据

```javascript
const timer1$ = timer(0, 1000);
const timer2$ = timer(500, 1000);
withLatestForm(timer1$, timer2$).subscribe(console.log);
// output: [1, 0], [2, 1]...
```

**race** :类似Promise.race，仅处理最先到达的Observable

```javascript
const timer1$ = timer(0, 1000);
const timer2$ = timer(500, 1000);
race(timer1$, timer2$).subscribe(console.log);
// 仅处理由timer$产出的Observable对象
```

**startWith** ： 在被订阅时先突出一定的数据

```javascript
const timer1$ = timer(0, 1000).startWith('start');
const timer2$ = timer(500, 1000);
race(timer1$, timer2$).subscribe(console.log);
// output: start 0 1 2
```

**forkJoin** : 等待所有Observable对象处理完结，选取每个最后一个对象传递给下游。

```javascript
const timer1$ = timer(0, 1000).take(1);
const timer2$ = timer(500, 1000).take(3);
forkJoin(timer1$, timer2$).subscribe(console.log);
//output: 0, 2
```

**switch**： 总是切换到最新的内部Observable对象获取数据；

```javascript
const stream$ = Observable.interval(1000).take(2).map(v => Observable.interval(1500).map(v => `${x},${y}`).take(2));
const result = stream$.switch();
// output: 1,0 1,1
```

**exhaust** : 耗尽当前Observable之前不会切换到下一个；

```javascript
const stream$ = Observable.interval(1000).take(2).map(v => Observable.interval(1500).map(v => `${x},${y}`).take(2));
const result = stream$.exhaust();
// output: 0,0 0,1
```



## 辅助操作符

| 功能                                     | 操作符          |
| ---------------------------------------- | --------------- |
| 统计数据流中所有数据的个数               | count           |
| 获得数据流中最大和最小的数据             | Max, min        |
| 对数据流传输数据进行规约操作             | reduce          |
| 判断是否所有数据满足判断条件             | every           |
| 找到的一个满足判定条件的数据             | Find, findIndex |
| 判断一个数据流是否不包含任何数据         | isEmpty         |
| 如果一个数据流为空就默认产生一个指定数据 | defaultIfEmpty  |

**count** ： 统计上游Observable对象吐出的所有数据。

```javascript
Observable.timer(1000).concat(Observable.timer(100)).count();
//output: 2
Observable.timer(1000).concat(Observable.timer(100), Observable.timer(100)).count();
// output: 3
```

**max， min** ： 获得Observable对象的最大值和最小值。

```javascript
Observable.of(1,2,3).min((a,b) => a - b)
// output: 1
```

**reduce** : 规约统计，类似于JavaScript的数组方法reduce。

```javascript
Rx.Observable.of(1,2,3).reduce((acc, curr) =>acc + curr).subscribe(console.log);
//output: 6
```

**every** ：类似数组方法every，当所有的Observable对象满足某个条件时，返回true，否则false；

```javascript
Rx.Observable.of(1,2,3).every((x) => x > 2)
// output: false
```

**find，findIndex** ：类似数组方法；

```javascript
Rx.Observable.of(1,2,3).find((x) => x > 2).subscribe(console.log);
// output: 3
Rx.Observable.of(1,2,3).findIndex((x) => x > 2).subscribe(console.log);
// output: 2
```

**isEmpty** ：

```javascript
Rx.Observable.create(observer => {
    setTimeout(() => observer.complete(1), 1000);
}).isEmpty().subscribe(console.log);
// output: 一秒后。。 true
```

**defaultIfEmpty** ：

```javascript
Rx.Observable.create(observer => {
    setTimeout(() => observer.complete(1), 1000);
}).defaultIfEmpty().subscribe(console.log);
// output: 一秒后。。 null
```



## 	过滤数据流

| 功能                               | 操作符                                        |
| ---------------------------------- | --------------------------------------------- |
| 过滤掉不满足条件的操作符           | filter                                        |
| 获得满足条件的第一个数据           | first                                         |
| 获得满足判断条件的最后一个条件     | last                                          |
| 从数据流中取出最先出现的若干年数据 | take                                          |
| 从数据流中取出最后出现的若干数据   | takeLast                                      |
| 从数据流中选取数据直到某种情况发生 | takeWhile，takeUntil                          |
| 从数据流中忽略最先出现的若干数据   | skip                                          |
| 从数据流中忽略数据直到某种情况发生 | skip， skipUntil                              |
| 基于时间的数据流量筛选             | throttleTime, debounceTime, auditTime         |
| 基于数据内容的数据流量筛选         | throttle，debounce，audit                     |
| 基于采样方式的数据流量筛选         | sample，sanpleTime                            |
| 删除重复的数据                     | distinct                                      |
| 删除重复的连续数据                 | distinceUntilChanged， distinctUntilKyChanged |
| 忽略数据流中的所有数据             | ignoreElements                                |
| 只选取指定出现位置的数据           | elementAt                                     |
| 判断是否只有一个数据满足判定条件   | single                                        |

**filter** ：类似数组的filter方法，过滤掉不满足条件的数据。

```javascript
Observable.of(1,2,3).filter(v => v%2 == 0).subscribe(console.log);
// output: 2
```

**first** : 满足条件的第一个数据。

```javascript
Observable.of(1,2,3).first(v => v%3 == 0).subscribe(console.log);
// output: 3
Observable.of(1,2,3).first().subscribe(console.log);
// output: 1
```

**last** ： 与first相反。

```javascript
Observable.of(1,2,3).last(v => v%3 == 0).subscribe(console.log);
// output: 3
Observable.of(1,2,3).last().subscribe(console.log);
// output: 3
```

**take，takeLast，takeWhile， takeUntil** ： 从数据流中取出多个数据，或按照条件中取出多个数据。

```javascript
Observable.of(1,2,3，4).take(3).subscribe(console.log);
// output: 1，2，3
Observable.of(1,2,3).takeLast(3).subscribe(console.log);
// output: 2,3,4
Observable.of(1,2,3).takeWhile(x => x < 2).subscribe(console.log);
// output: 1
Observable.interval(1000).takeUntil(Observable.interval(2500)).subscribe(console.log);
// 需要另行订阅高阶Observable
// output: 0,1

```

**skip ** ：跳过前n个

```javascript
Observable.of(1,2,3).skip(1).subscribe(console.log);
// output: 2,3
```

**skipWhile, skipUntil**

```javascript
Observable.interval(1000).skipWhile(v => v%2 == 0).subscribe(console.log)
// output: 1,2,3,4,5,6.....
Observable.interval(1000).takeUntil(Observable.interval(2500)).subscribe(v => console.log("I'm in v", v))
// output 1,2,3,4
// takeUntile skipUntil 行为相反
```



**throttleTime** ： 在限定时间范围内，从上游向下游传递数据的个数。

```javascript
Observable.interval(1000).throttleTime(2000).subscribe(console.log);
// output: 0, 2, 4, 6.....
```

**debounceTime** : 让传递给下游数据间隔不能够小雨给定时间。

```javascript
Observable.interval(1000).filter(v => v % 3 === 0).debounceTime(2000).subscribe(console.log);
// output: 0, 3, 6.....
```

**throttle** : 根据duorationSelector触发的时机来传递上游的值。

```javascript
Observable.interval(1000).throttle(v => {
          console.log("v", v);
          return Observable.interval(2000);
        }).subscribe(console.log)
// output: v0 0 , v2 2.....
```

**debounce** : 根据duorationSelector触发的时机来传递上游的值。

```javascript
Observable.interval(1000).debounce(v => {
          console.log("v", v);
          return Observable.interval(2000);
        }).subscribe(console.log)
// output: 时间间隔总有数据产生所以永远不会打印数据

```

**auditTime， audit** ： 与throttle类似但是传递的是最后一个数据。

```javascript
Observable.interval(1000).auditTime(3000).subscribe(console.log);
// output: 2, 5, 8,10
```

**sample, sampleTime**: 根据规则在一个范围内取一个数据抛弃其他数据。

```javascript
Observable.interval(1000).sampleTime(3000).subscribe(console.log);
// output： 1,4,7,10,13...
// sample 会在接收到notifer$产生数据后采集最后一个数据传递给下游
```

**distinct**： 只返回从没有出现过的数据。

```javascript
Observable.of(0,1,2,3,3,4,5,5,6).distinct().subscribe(console.log);
// output: 0,1,2,3,4,5,6
// @params func rule
// @params Observable flush 
// 理解为去重
```

**distinctUntilChanged， distinctUntilKeyChanged** ： 删除掉和上一个相同的数据。

```javascript
Observable.of(0,1,2,3,3,4,5,5,6,1).distinctUntilChanged().subscribe(console.log);
// output: 0,1,2,3,4,5,6,1
// @params func rule 判断对象相等的规则
// 理解为去重
```

## 转化操作符

| 功能                             | 操作符                                                 |
| -------------------------------- | ------------------------------------------------------ |
| 将每个元素映射函数产生新的数据   | map                                                    |
| 将数据流中每个元素映射为同一数据 | mapTo                                                  |
| 提取数据流中每个数据的某个字段   | pluck                                                  |
| 产生高阶ObservabTime,bule对象    | windowTime, windowCount,windowWhen,windowToggle,window |
| 产生数组构成的数据流             | bufferTime,bufferCount,bufferWhen,bufferToggle,buffer  |
| 映射产生高阶Observable对象并合并 | concatMap, mergeMap, switchMap, exhaustMap             |
| 产生规约运算结果组成的数据流     | scan， mergeScan                                       |

**map**

```javascript
const func = function(value) {
    return [value, this.cont]
}
const context = {cont: 'test'};            
Observable.of(1,2,3).map(func, context).subscribe(console.log);
// output: [0, 'test'] [1, 'test'] [2, 'test']           
```

**pluck** : 将上游的数据按字段取出。

```javascript
Observable.of({name: 'mark'}).pluck('name').subscribe(console.log);
// output: mark
```

**windowTime, bufferTime**

```javascript
      Observable.interval(1000).windowTime(2000).subscribe(v => v.subscribe(console.log));
// output: 1,2,3,4...
// windowToggle //以source为数据源，第一个Observable为凭据，进行Toggle操作
```



**window， buffer**

```javascript
const source$ = Observable.timer(0, 100);
const notifer$ = Observable.timer(400, 400);
source$.window(notifer$);
// 每400毫秒产生一个Observable的窗口
```

**concatMap, mergeMap, switchMap, exhaustMap** 

```javascript
Observable.of(1,2,3).concatMap(v => Observable.interval(100).take(3)).subscribe(console.log);
// output: 012012012
Observable.of(1,2,3).mergeMap(v => Observable.interval(100).take(3)).subscribe(console.log);
// output: 001122
Observable.of(1,2,3).switchMap(v => Observable.interval(100).take(3)).subscribe(console.log);
// output: 012
```

**groupBy**

```javascript
Observable.of(1,2,3,4).groupBy(v => v %2 === 0).subscribe((v) => v.subscribe(console.log));
// output: 2 , 4
// return Observable
```

**scan** ： 可以理解为可以持续传递数据的reduce

**mergeScan** ： 返回Observable的scan

## 异常处理

| 功能                         | 操作符           |
| ---------------------------- | ---------------- |
| 捕获上游的错误               | catch            |
| 当上游产生错误时重试         | retry，retryWhen |
| 无论是否出错都要进行一些操作 | finally          |

**catch**

```javascript
Observable.of(1,2,3,4).map((v) => {
          if(v == 4) {
            throw new Error('test');
          }
          return v;
        }).catch((err, caught$) => {
          console.log("test", err);
          return caught$;
        }).take(10).subscribe(console.log);
// output: 1,2,3,Error: test
```

**retry, retryWhen**

```javascript
Observable.of(1,2,3,4).map((v) => {
          if(v == 4) {
            throw new Error('test');
          }
          return v;
        }).retry(1).catch((err, caught$) => {
          return Observable.of(8);
        }).subscribe(console.log);
// output: 1,2,3,1,2,3,8

Observable.of(1,2,3,4).map((v) => {
          if(v == 4) {
            throw new Error('test');
          }
          return v;
        }).retryWhen(err$ => Observable.interval(10)).catch((err, caught$) => {
          return Observable.of(8);
        }).subscribe(console.log);
// output: 123123123123123123123....
```

**finally** : 同Javascript的finally方法。

## 多播

| 功能                             | 操作符          |
| -------------------------------- | --------------- |
| 灵活选取Subject对象进行多播      | multicast       |
| 只多播数据流中最后一个数据       | publishLast     |
| 对数据流中给定数量的数据进行多播 | publishReplay   |
| 拥有默认数据的多播               | publishBehavior |

> Cold Observable: 每次subscribe都产生一个全新的数据序列的数据流。
>
> Hot Observable： 真正的数据源与Observer没有关系，达到统一数据源的效果（fromEvent, fromPromise）。

```javascript
const subject = new Subject();
const subjectTest = subject.subscribe((v) => console.log('v1',v));
subject.next(1);
subject.subscribe(v => console.log('v2', v), null, ()=> console.log('v2 complete'));
subject.next(2);
subjectTest.unsubscribe();
subject.complete();
// output: v1,1 v1,2 v2,2 v2complete
```

> subject可以有多个数据源，起到的作用就是把多个数据源的内容汇总到一个Observable中去。

**multicast**

```javascript
const tick$ = Observable.interval(1000).take(10);
const source = tick$.multicast(new Subject()).refCount();
source.subscribe(v => console.log("v1", v))
source.subscribe(v => console.log("v2", v))
// output: v1 0, v2 0 v1 1, v2 1....
// multicast产生的connectableObservable会去subscribe的Observable
const tick$ = Observable.interval(1000).take(2);
const source = tick$.multicast(new Subject(), (shared) => shared.concat(Observable.of('Done')))
source.subscribe(v => console.log("I'm in v1", v))
source.subscribe(v => console.log("I'm in v", v))
// I'm in v1 0
// I'm in v 0
// I'm in v1 0
// I'm in v 0
// I'm in v1 1
// I'm in v 1
// I'm in v1 Done
//'m in v Done

```

**publish**

```javascript
const publish = (selector) => {
    if(selector){
        return this.multicast(new Subject(), selector)
    }
    return this.multicast(new Subject())
}
// publish实际是对multicast的封装
```

**share**

```javascript
const share = () {
    this.multicast(() => new Subject()).refcount();
}

// share也是对multicast的封装
```

**publishLast，AsyncSubject**

```javascript
function publishLast() {
    return this.multicast.call(this, new AsyncSubject())
}
// 由AsyncSubject封装的publishLast方法
```

**publishReplay， replaySubject**

```javascript
function publishReplay(bufferSize = Number.POSITIVE_INFINITY, windowTime = Number.POSITIVE_INFINITY) {
    return multicast.call(this, new ReplaySubject(bufferSize, windowTime))
}
```

**publishBehavior, BehaviorSubject**

```javascript
function publishBehavior(value) {
    return multicast.call(this, new BehaviorSubject(value));
}
```



## Scheduler

> Scheduler 可以作为创造类和合并类操作符的函数使用，Rx还提供了observeOn和subscribeOn两个操作符，用于在管道任何位置插入Scheduler

```
console.log('before');
Observable.range(1, 3).subscribeOn(asap).subscribe(console.log)
console.log('after');
// output: 1,2,3,before, after
```



















