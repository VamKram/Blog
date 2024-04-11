title: Webpack KING！！ (二)
date: 2019/08/15
cover: https://raw.githubusercontent.com/webpack-contrib/awesome-webpack/master/media/awesome_webpack_branding.png
categories:
- tool
tags:
- webpack

---

# webpack king（二）

## 关于Tapable

>  Webpack的插件系统是基于事件的，通过发布订阅模式构建webpack的事件系统。

tapable是什么以及简单的实现。

```javascript
let { SyncHook, AsyncParallelHook } = require("tapable");
class Test {
    constructor() {
        this.hooks= {
            arch: new SyncHook(['name'])
        }
    }
    tap() {
        this.hooks.arch.tap("test1", (...args) => {
            console.log(args)
        });
        this.hooks.arch.tap("test2", (...args) => {
            console.log(args)
        });
    }
    start() {
        console.log(111)
        this.hooks.arch.call('111');
    }
}


class Test1 {
    constructor() {
        this.hooks= {
            arch: new AsyncParallelHook(['name'])
        }
    }
    tap() {
        this.hooks.arch.tapAsync("test1", (...args, cb) => {
            setTimeout(() => {
              console.log(123)
            }, 1000)
        });
        this.hooks.arch.tapAsync("test2", (...args, cb) => {
           setTimeout(() => {
              console.log(123)
            }, 1000)
        });
    }
    start() {
        console.log(111)
        this.hooks.arch.callAsync('111');
    }
}
```

AsyncParallelHook

```javascript
class AsyncParallelHook {
  constructor(){
    this.tasks = [];
  }
  callAsync(...args) {
    let fn = args.pop();
    let idx = 0
    function done(name, cb) {
      idx++;
      if (idx === this,tasks.length) {
        fn();
      }
    }
    this.tasks.forEach(t => {
      task(...args, done)
    })
   }
  tapPromise(name, cb) {
    this.tasks.push(cb)
  }
  promise(...arg) {
        return Promise.all(this.tasks.map(t => t(arg)))
  }
  tapAsync(name, cb) {
    this.tasks.push(cb)
  }
}


```

AsyncSeries

```javascript
class AsyncSeries {
  constructor(){
    this.tasks = [];
  }
  callAsync(...args) {
    let fn = args.pop();
    let idx = 0;
    const next = () => {
      if (this.tasks.length === idx) return fn()
      let tsk = this.tasks[index++];
      tsk(...arg, next);
    }
    next()
   }

  tapAsync(name, cb) {
    this.tasks.push(cb)
  }
}
```

AsyncSeriesWaterfallHook

```javascript
class AsyncSeriesWaterfallHook {
  constructor(){
    this.tasks = [];
  }
  callAsync(...args) {
    let fn = args.pop();
    let idx = 0;
    const next = (error, data) => {
      let tsk = this.tasks[index];
      if (!tsk) return fn()
      tsk(...arg, next);
    }
    index === 0 && task(...args, next)
    index !== 0 && task(data, next)
   }

  tapAsync(name, cb) {
    this.tasks.push(cb)
  }
}
```

