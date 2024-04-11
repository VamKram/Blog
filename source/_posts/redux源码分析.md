---
title: redux 源码分析
date: 2018/03/25
cover: http
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

![image-20210310235527758](http

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
```

以下是我用ts实现的版本

```typescrip

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
### react-redux

redux是框架无关的一个库，要想在react中使用就必须借助于react-redux，而结合的关键就在于 react-redux的
connect的方法

```javacript
/**
* mapStateToProps(state, ownProps) : stateProps
* mapDispatchToProps(dispatch, ownProps) : dispatchProps
*/
connect(mapStateToProps, mapDispatchTOProps, mergeProps, options)
```

```javascript
class Provider extends Component {
  constructor(props) {
    super(props)
    const { store } = props
    this.notifySubscribers = this.notifySubscribers.bind(this)
    const subscription = new Subscription(store)的subscrption对象上的更新函数
    subscription.onStateChange = this.notifySubscribers
    this.state = {
      store,
      subscription
    }
    this.previousState = store.getState()
  }

  componentDidMount() {
    this._isMounted = true

    this.state.subscription.trySubscribe()

    if (this.previousState !== this.props.store.getState()) {
      this.state.subscription.notifyNestedSubs()
    }
  }

  componentWillUnmount() {
    if (this.unsubscribe) this.unsubscribe()
    this.state.subscription.tryUnsubscribe()
    this._isMounted = false
  }

  componentDidUpdate(prevProps) {
    if (this.props.store !== prevProps.store) {
      this.state.subscription.tryUnsubscribe()
      const subscription = new Subscription(this.props.store)
      subscription.onStateChange = this.notifySubscribers
      this.setState({ store: this.props.store, subscription })
    }
  }

  notifySubscribers() {
    this.state.subscription.notifyNestedSubs()
  }

  render() {
    const Context = this.props.context || ReactReduxContext
    return (
      <Context.Provider value={this.state}>
        {this.props.children}
      </Context.Provider>
    )
  }
}
```

### 工作流程
1. get store
2. 使用发布订阅，notify所有的subscriber
3. didmount阶段如果有更新重新赋值
4. 检查store是否equal否则 重新2
5. 传递context

```javascript
subscribe(listener) {
      let isSubscribed = true
      if (next === current) next = current.slice()
      next.push(listener)

      return function unsubscribe() {
        if (!isSubscribed || current === CLEARED) return
        isSubscribed = false
        if (next === current) next = current.slice()
        next.splice(next.indexOf(listener), 1)
      }
    }
  }

// 这里订阅的是一个 wrapper
  trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper)
      this.listeners = createListenerCollection()
    }
  }
// state or dispatch =》 props
  function selector(stateOrDispatch, ownProps) {
  return props
}
```
> 总的来说: react-redux做的就是 将provider的value放入context，connect阶段取出store获取 mapStateToProps， mapDispatchTOProps，然后通过select create props传入组件并且通过一个subscripation 订阅当前state的dependency，通过react unstable_batchUpdate 来做差量的更新

