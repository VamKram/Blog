---
title: Redux Middleware笔记
date: 2017/07/19
categories:
- redux
tags: 
- redux

---
# Redux Middleware笔记

> 最近囿于公司React，Redux项目越来越繁杂，经手的人也越来越多，就想通过redux middleware的方式对action，reducer的性能进行监控，以达到控制代码质量的目的，middleware的代码逻辑非常简单，这次希望从更深的方面去了解Redux Middleware的工作机制。

#### React，Redux的工作原理

Redux遵循着三大理念

+ Single source of truth(单一数据源)
+ State is readOnly（状态只读）
+ Changes are made with pure functions（使用[纯函数](https://medium.freecodecamp.org/why-redux-needs-reducers-to-be-pure-functions-d438c58ae468)来修改Store)



Redux，React，ReduxMiddleWare三者在项目中有一个交汇点，那就是CreateStore方法中enchancer参数接受的applyMiddleware方法。

```typescript
export default function createStore(reducer, preloadedState, enhancer) {
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }
//。。。。
    

```

以下是ApplyMiddleware的源码：

```typescript
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    // 通过对middle的规约传入dispatch
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```



```typescript
// 这个函数非常的有意思，它的作用就是可以把func以高阶函数的形式组织起来
// compose(funcA, funcB, funcC) => funcA(funcB(funcC))
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```



进过compose的组织applyMiddleware的工作机制已经清楚了，简单来说applymiddleware方法在调用middleware时形如：

```javascript
	middleware(store)(dispatch);
```

所以当我们编写middleware时只需要基于准则就可以编写一个方便易用的middleware了

```
func = (store) => dispatch => {
	dispatch(action)
} 

```

比如我们想要检测action的性能

```
export const spendTimeLog = store => next => action => {
      console.log( '%c emit ', 'background: #e9e6e7; color: #888', action );
	  var start = new Date().getTime();
	  const result = next( action );
      var end = new Date().getTime();
      console.log( `%c result "${action.type}" spend ${ end-start } milliseconds.`, 'background: #e9e6e7; color: #000' );
      return result;
};

export default perflogger;
```

