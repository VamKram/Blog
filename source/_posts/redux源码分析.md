---
title: redux 源码分析
date: 2018/03/25
cover: https://technologybook.tech/assets/img/r5.png
categories:
- react redux source
tags:
- redux source

---

# Redux源码分析

> Hope is not lost today... it is found
>
> -leia

redux的实现是非常简洁的，本文旨在通过对源码的解读和简易实现加深对code的理解。

![image-20210310235527758](https://technologybook.tech/assets/img/r5.png)

源码的目录结构非常清晰，文件名即是API的名字。

### createStore

```typescript
/**
 * Creates a Redux store that holds the state tree.
 * The only way to change the data in the store is to call `dispatch()` on it.
 *
 * There should only be a single store in your app. To specify how different
 * parts of the state tree respond to actions, you may combine several reducers
 * into a single reducer function by using `combineReducers`.
 *
 * @param {Function} reducer A function that returns the next state tree, given
 * the current state tree and the action to handle.
 *
 * @param {any} [preloadedState] The initial state. You may optionally specify it
 * to hydrate the state from the server in universal apps, or to restore a
 * previously serialized user session.
 * If you use `combineReducers` to produce the root reducer function, this must be
 * an object with the same shape as `combineReducers` keys.
 *
 * @param {Function} [enhancer] The store enhancer. You may optionally specify it
 * to enhance the store with third-party capabilities such as middleware,
 * time travel, persistence, etc. The only store enhancer that ships with Redux
 * is `applyMiddleware()`.
 *
 * @returns {Store} A Redux store that lets you read the state, dispatch actions
 * and subscribe to changes.
 */

```



通过函数签名可以看出createStore包含哪些信息

```javascript
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

  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }
```

这一部分主要看对enhancer的设计

因为createStore创建store的时候主要有以下几种形式

- createStore(reducer, initialState)
- createStore(reducer)
- createStore(reducer, enhancer)
- createStore(reducer, initialState, enhancer)

在代码中都分别做了防御性的编写，进行了params参数的交接



### compose

一个非常有趣的函数 用于把函数改造成 **a(b(c()))**嵌套调用的形式。



```javascript
function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```



同理的还有pipe

```javascript
function pipe(...funcs) {
  return (args) => funcs.reduce((acc, cur) => cur(acc), args);
}	

// pipe(funcA, funcB, funcC)(args)
```

以下是我用ts实现的版本

```typescript
// const store = createStore(reducer, enhancer);
// store.dispatcher({type: string})

type IAction = { type: string | symbol, payload?: any }
type IReducer<S extends object> = (state: S, action: IAction) => S;
type ISyncAction<S extends object> = (
    getState: Store<S>["getState"],
    dispatch: Store<S>["dispatch"]) => void;
type IApplyMiddleware<S extends object> = (store: Store<S>) => Store<S>
interface Store<S extends object> {
    getState: () => S;
    subscribe: (listener: Function) => () => void;

    dispatch(params: IAction | ISyncAction<S>): void;
}

const noop = () => {
};

export function createStore<S extends object = {}>
(
    reducer: IReducer<S>,
    initialState?: S,
    applyMiddleware?: IApplyMiddleware<S>
): Store<S> {
    let subscriptions: Function[] = [];
    let onSubscribe: boolean = false;
    let state = initialState || {} as S;
    if (applyMiddleware) {
        return applyMiddleware(createStore(reducer, initialState));
    }
    function getState() {
        return state;
    }

    function dispatch(params: IAction | ISyncAction<S>) {
        state = reducer(state, params as IAction);
        onSubscribe && subscriptions.forEach(f => f());
    }

    function subscribe(listener: Function) {
        if (typeof listener !== "function") {
            console.error("you subscribe a function!");
            return noop;
        }
        onSubscribe = true;
        subscriptions.push(listener);
        return function unsubscribe() {
            onSubscribe = false;
            subscriptions.length = 0
            console.log('unsubscribe done!');
        }
    }

    return {
        getState,
        dispatch,
        subscribe
    }
}

function compose(...fn) {
    if (fn.length === 0) {
        return args => args;
    }
    if (fn.length === 1) {
        return fn[0]
    }
    return fn.reduce((f1, f2) => (...args) => f1(f2(args)));
}


function applyMiddleware(...middlewares) {
    return store => {
        let dispatch = store.dispatch;
        let getState = store.getState;
        const p = {getState, dispatch}
        const middlewaresWithApi = middlewares.map(md => md(p));
        dispatch = compose(middlewaresWithApi)(dispatch);
        return {
            ...store,
            dispatch
        }
    }
}

export function combineReducers<S extends object>(reducers) {
    const combinedReducerMap = {};
    Object.keys(reducers).forEach(key => {
        if (typeof reducers[key] === "function") {
            combinedReducerMap[key] = reducers[key];
        }
    });

    return function(state = {}, action: IAction) {
        const nextState = {};
        let hasChanged = false;
        Object.keys(combinedReducerMap).forEach(key => {
            const reducer = combinedReducerMap[key];
            const preState = state[key];
            const currentState = reducer(preState, action);
            if (typeof currentState === void 0) {
                console.error("key is not exist!");
            }

            nextState[key] = currentState;
            hasChanged = hasChanged || currentState !== preState;
        });
        return hasChanged ? nextState : state;
    };
}

```

