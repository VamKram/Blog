---
title: 垒积木之实现一个MiniReact
date: 2019/12/25
cover: https://technologybook.tech/assets/img/3.png
categories:
- react
tags:
- react source

---

![rs](https://technologybook.tech/assets/img/3.png)

# react source code

## 屏幕刷新

> 屏幕刷新率一般是 16.6ms

#### 一个完整的帧的渲染过程

1. input events（Blocking input events(touch/scroll) / non-blocking input event(click/keypress)）
2. timer
3. 每一帧执行的事件(resize/scroll/media query change)
4. rAF（绘制前执行）
5. layout(recalculate style/update layout)
6. paint(compositing/paint invalidation/record)
7. Idle (idle callback)

![image-20210304153048312](https://technologybook.tech/assets/img/3.png)

> ***v8执行js和page render 是在同一个渲染线程，因此GUI渲染和js执行是互斥的。***

#### API: requestAnimationCallback / requestIdleCallback

> `requestAnimationFrame`: 的时间间隔与屏幕刷新率相关，rAF可以保证每一帧都只执行一次，且每一次执行的时机最优为屏幕刷新一次的时间（60fps 则为 16.6ms）；
>
> `requestIdleCallback`: **检测浏览器空闲并执行任务。**（判断是否耗时大于当前帧执行时间，有空余即执行）
>
> 1. rIC像浏览器申请时间片
> 2. 等待渲染布局绘制资源加载事件执行完毕
> 3. 分配时间片 (初始化为5ms)
>
> ```javascript
> 
>   // Scheduler periodically yields in case there is other work on the main
>   // thread, like user events. By default, it yields multiple times per frame.
>   // It does not attempt to align with frame boundaries, since most tasks don't
>   // need to be frame aligned; for those that do, use requestAnimationFrame.
>   let yieldInterval = 5;
>   const maxYieldInterval = 300;
>   let needsPaint = false;
> ```
>
>
>
> ​ 4. 归还控制权
>
> 若浏览器一直繁忙，`requestIdleCallback` 注册的任务有可能一直挂起。可通过设置 `timeout`来保证超时执行。



---

## Fiber

1. 为什么是fiber?

> 为了弥补老架构基于堆栈递归不可中断导致DOM渲染不完全的问题，创造出可以中断更新在恢复的Fiber架构。

2. 为什么不是generator function？

> generator会污染其他函数，彻底改变框架的使用方式,导致使用成本的增加。
>
> v16任务调度优先级不能很好的实现 [原因](https://github.com/facebook/react/issues/7942#issuecomment-254987818)



![Untitled Diagram ](https://technologybook.tech/assets/img/5.png)

## fiber包括什么及其作用

1. **Tag/type/props**: 等描述一个组件如何在页面渲染的信息。
2. **effectTag/lastEffect / fisrtEffect / nextEffect**:  所组成的 Effect List 收集进行过副作用操作的fiber最终遍历更新。
3. **hooks/updateQuene**: 在组件的中的setState和hooks的更新方式
4. **expirationTime**: 任务更新优先级
5. **alternate**: 利用双缓存进行优化更新的手段currentFiber与alternate指向的oldFiber比较（diff）。
6. **child/return/sibling**: fiber的基本结构

### reconciliation

> 老架构: React递归的对比vdom，通过diff然后更新，在此期间js一直执行，阻塞用户操作的响应，产生卡顿的感觉。
>
> 问题: 不可中断，执行栈过深时的性能问题。
>
> 新架构: 基于fiber的可中断的reconciliation，react会在空闲时间片执行任务,解决了网络IO带来的用户体验问题。

> v15是基于堆栈的，Reconciler。先 mountComponent后updateComponennt递归向下执行。
>
> v16是链表二叉树通过EffectTag确定TAG然后对有Effect的fiber进行相应的操作统一处理完成后交给renderer处理

```typescript
export interface Fiber {
    root?: null;
    child?: IRoot;
    sibling?: IRoot;
    return?: IRoot;
    firstEffect?: IRoot | null,
    lastEffect?: IRoot | null,
    nextEffect?: IRoot | null
}

export interface IRoot extends Fiber {
    tag: symbol;
    type: string | keyof HTMLMapElement | Component<any> | Function;
    props: {
        children: ReactElement[],
        text?: string
    } & { [K in string]: any },
    node: null | HTMLElement | Component;
    return?: IRoot,
    alternate?: IRoot,
    hooks?: any
    effectTag?: symbol | null,
    updateQueue?: UpdateQueue,
    ref?: Ref
    expirationTime: number
}
```

### vdom

> 通过js描述页面元素的方式 React.creatELement

### 执行过程

1. 从根节点的渲染调度
    1. diff diff两个vdom进行增量的更新创建，基于rIC实现可以暂停的比较，更新。
    2. render 生成fiber, 收集产生的side effect
    3. firstEffect 指向第一个有effect的fiber lastEffect指向最后一个有副作用的child fiber
    4. commit dom创建不能暂停

2. 调度优先级 expirationTime

### 双缓存机制

![123](https://technologybook.tech/assets/img/4.png)

```typescript
function createWorkInProgress(current, pendingProps) {
    var workInProgress = current.alternate;
    if (workInProgress === null) {
        workInProgress = createFiber(current.tag, pendingProps, current.key, current.mode);
        workInProgress.elementType = current.elementType;
        workInProgress.type = current.type;
        workInProgress.stateNode = current.stateNode;
        workInProgress.alternate = current;
        current.alternate = workInProgress;
    }

```

```typescript
    // 第二次的更新
if (currentRoot && currentRoot.alternate) {
    workInProgressRoot = currentRoot.alternate;
    workInProgressRoot.alternate = currentRoot;
    if (fiberRoot) workInProgressRoot.props = fiberRoot.props;

    // 第一次更新
} else if (currentRoot) {
    if (fiberRoot) {
        fiberRoot.alternate = currentRoot;
        workInProgressRoot = fiberRoot;
    } else {
        workInProgressRoot = {
            ...currentRoot,
            alternate: currentRoot,
        }
    }
} else {
    workInProgressRoot = fiberRoot;
}
if (workInProgressRoot) {
    workInProgressRoot.firstEffect = workInProgressRoot.lastEffect = workInProgressRoot.nextEffect = null;
}
nextUnitOfWork = fiberRoot;
```

Fiber上的`alternate`属性是双缓存（bouble buffering）的关键，它指向一个 Fiber，创建 WorkInProgress 节点时优先取`alternate`，没有的话就创建一个新的。

WorkInProgress是渲染到屏幕上fiber的最终状态

创建 WorkInProgress Tree 的过程是一个 Diff 的过程，Diff的结果是一个Effect list，

effect list 最终会生成一个 effect的单链表，在commit阶段会迭代这个链表，通过effectTag 来决定如何UPDATE进而确定如何处理dom。

### beginwork

> 分配到时间片后，从fiberroot开始后序遍历fiber tree，在此阶段创建fiber。

```typescript
// 构建fiber阶段

function updateHostRoot(fiber: IRoot) {
    let newChildren = fiber.props.children;
    reconcileChildren(fiber, newChildren);
}

function updateDOM(stateNode: HTMLElement, oldProps: IRoot["props"], newProps: IRoot["props"]) {
    setProps(stateNode, oldProps, newProps);
}

function createDOM(fiber: IRoot): HTMLElement | null {
    if (fiber.tag === ReaxTypes.NODE_TEXT) {
        return document.createTextNode(fiber.props?.text || "") as any;
    }
    if (fiber.tag === ReaxTypes.NODE_HOST) {
        let stateNode = document.createElement(fiber.type as keyof HTMLElementTagNameMap);
        updateDOM(stateNode, {} as any, fiber.props);
        return stateNode;
    }
    return null;
}

function updateHostText(fiber: IRoot) {
    if (!fiber.node) {
        fiber.node = createDOM(fiber)
    }
}

function updateHost(fiber: IRoot) {
    if (!fiber.node) {
        fiber.node = createDOM(fiber)
    }
    const newChildren = fiber.props.children;
    reconcileChildren(fiber, newChildren);
}

function beginWork(fiber: IRoot) {
    if (fiber.tag === ReaxTypes.ROOT) {
        updateHostRoot(fiber);
    } else if (fiber.tag === ReaxTypes.NODE_TEXT) {
        updateHostText(fiber);
    } else if (fiber.tag === ReaxTypes.NODE_HOST) {
        updateHost(fiber);
    } else if (fiber.tag === ReaxTypes.NODE_CLASS) {
        updateClassComp(fiber)
    } else if (fiber.tag === ReaxTypes.NODE_FUNCTION) {
        updateFunctionComp(fiber)
    }
}
```

## completework

```typescript
// 副作用收集
function completeUnitOfWork(nextUnitOfWork: IRoot) {
    let returnFiber = nextUnitOfWork.return;
    if (returnFiber) {
        if (!returnFiber.firstEffect) {
            returnFiber.firstEffect = nextUnitOfWork.firstEffect;
        }
        if (nextUnitOfWork.lastEffect) {
            if (returnFiber.lastEffect) {
                returnFiber.lastEffect.nextEffect = nextUnitOfWork.firstEffect;
            }
            returnFiber.lastEffect = nextUnitOfWork.lastEffect;
        }
        const effectTag = nextUnitOfWork.effectTag
        if (effectTag) {
            if (returnFiber.lastEffect) {
                returnFiber.lastEffect.nextEffect = nextUnitOfWork;
            } else {
                returnFiber.firstEffect = nextUnitOfWork;
            }
            returnFiber.lastEffect = nextUnitOfWork;
        }
    }
}
```

## commitWork

```typescript
function commitWork(fiber: IRoot) {
    if (!fiber) return;
    let retFiber = fiber.return as IRoot;
    while (retFiber.tag !== NODE_HOST && retFiber.tag !== ROOT && retFiber.tag !== NODE_TEXT) {
        retFiber = retFiber.return as IRoot;
    }
    let retDOM = retFiber.node as HTMLElement;
    if (fiber.effectTag === ReaxTypes.PLACEMENT && retDOM) {
        let nextFiber = fiber;
        if (nextFiber.tag === NODE_CLASS) return;
        while (nextFiber.tag !== NODE_HOST && nextFiber.tag !== NODE_TEXT) {
            nextFiber = fiber.child as IRoot;
        }
        retDOM.appendChild(nextFiber.node as HTMLElement);
    }
    if (fiber.effectTag === ReaxTypes.DELETION && retDOM) {
        return commitDeletion(fiber, retDOM);
    }
    if (fiber.effectTag === ReaxTypes.UPDATE && retDOM) {
        if (fiber.type === ELEMENT_TEXT) {
            if (fiber.alternate?.props.text != fiber.props.text) {
                if (fiber.node) {
                    (fiber.node as HTMLElement).textContent = fiber.props.text as string;
                }
            }
        } else {
            if (fiber.tag === NODE_CLASS) {
                return
            }
            fiber.node && updateDOM((fiber.node as HTMLElement), fiber.alternate?.props || {} as IRoot["props"], fiber.props);
        }
    }
    fiber.effectTag = null;
}
```

### Diff

1. Child 类型为`object`、`number`、`string`

> 比较key/type/tag相同确定是否有可服用的statenode，如果没有直接删除生成一个新的fiber节点

```javascript
function reconcileSingleElement (
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  expirationTime: ExpirationTime,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    // TODO: If key === null and child.key === null, then this only applies to
    // the first item in the list.
    if (child.key === key) {
      if (
        child.tag === Fragment
          ? element.type === REACT_FRAGMENT_TYPE
          : child.elementType === element.type
      ) {
        deleteRemainingChildren(returnFiber, child.sibling);
        const existing = useFiber(
          child,
          element.type === REACT_FRAGMENT_TYPE
            ? element.props.children
            : element.props,
          expirationTime,
        );
        existing.ref = coerceRef(returnFiber, child, element);
        existing.return = returnFiber;
        if (__DEV__) {
          existing._debugSource = element._source;
          existing._debugOwner = element._owner;
        }
        return existing;
      } else {
        deleteRemainingChildren(returnFiber, child);
        break;
      }
    } else {
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  if (element.type === REACT_FRAGMENT_TYPE) {
    const created = createFiberFromFragment(
      element.props.children,
      returnFiber.mode,
      expirationTime,
      element.key,
    );
    created.return = returnFiber;
    return created;
  } else {
    const created = createFiberFromElement(
      element,
      returnFiber.mode,
      expirationTime,
    );
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}

```

1. Child`newChild`类型为`Array`

> 因为有EffectTag，所以有限判断状态为UPDATE的节点再处理为DELECTION和PLACEMENT的节点。
>
> 比较方法 type/key不同则直接DELECTION

```javascript
function reconcileChildrenArray (
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  expirationTime: ExpirationTime,
): Fiber | null {
  // This algorithm can't optimize by searching from both ends since we
  // don't have backpointers on fibers. I'm trying to see how far we can get
  // with that model. If it ends up not being worth the tradeoffs, we can
  // add it later.

  // Even with a two ended optimization, we'd want to optimize for the case
  // where there are few changes and brute force the comparison instead of
  // going for the Map. It'd like to explore hitting that path first in
  // forward-only mode and only go for the Map once we notice that we need
  // lots of look ahead. This doesn't handle reversal as well as two ended
  // search but that's unusual. Besides, for the two ended optimization to
  // work on Iterables, we'd need to copy the whole set.

  // In this first iteration, we'll just live with hitting the bad case
  // (adding everything to a Map) in for every insert/move.

  // If you change this code, also update reconcileChildrenIterator() which
  // uses the same algorithm.

  if (__DEV__) {
    // First, validate keys.
    let knownKeys = null;
    for (let i = 0; i < newChildren.length; i++) {
      const child = newChildren[i];
      knownKeys = warnOnInvalidKey(child, knownKeys);
    }
  }

  let resultingFirstChild: Fiber | null = null;
  let previousNewFiber: Fiber | null = null;

  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }
    const newFiber = updateSlot(
      returnFiber,
      oldFiber,
      newChildren[newIdx],
      expirationTime,
    );
    if (newFiber === null) {
      // TODO: This breaks on empty slots like null children. That's
      // unfortunate because it triggers the slow path all the time. We need
      // a better way to communicate whether this was a miss or null,
      // boolean, undefined, etc.
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break;
    }
    if (shouldTrackSideEffects) {
      if (oldFiber && newFiber.alternate === null) {
        // We matched the slot, but we didn't reuse the existing fiber, so we
        // need to delete the existing child.
        deleteChild(returnFiber, oldFiber);
      }
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    if (previousNewFiber === null) {
      // TODO: Move out of the loop. This only happens for the first run.
      resultingFirstChild = newFiber;
    } else {
      // TODO: Defer siblings if we're not at the right index for this slot.
      // I.e. if we had null values before, then we want to defer this
      // for each null value. However, we also don't want to call updateSlot
      // with the previous one.
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  if (newIdx === newChildren.length) {
    // We've reached the end of the new children. We can delete the rest.
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }

  if (oldFiber === null) {
    // If we don't have any more existing children we can choose a fast path
    // since the rest will all be insertions.
    for (; newIdx < newChildren.length; newIdx++) {
```

### 节点移动的算法

1234 -> 1423

1. L(1) === R(1) equal => lastPlacedIndex = Idx(1) = 0;
2. L(2) !== R(4)  => oldIndex = Idx(4) = 3 > 0 => 不动 => lastPlacedIndex = 3

> if (oldIndex >= lastPlacedIndex) {
>
> 不动
>
> lastPlacedIndex = oldIndex
>
> } else {
>
> 移动到右边
>
> }

3. oldIndx(R(2)) = 1 < 3 => 右移
4. oldIndex(R(3)) = 2 < 3 => 右移

## commit

```typescript
function commitRoot() {
    deletions.forEach(commitWork);
    let fiber = workInProgressRoot?.firstEffect;
    while (fiber) {
        commitWork(fiber);
        fiber = fiber.nextEffect;
    }
    deletions.length = 0;
    currentRoot = workInProgressRoot;
    workInProgressRoot = null;
}

```

```javascript
    // 源码中的处理
// Check if work was scheduled by one of the effects
const rootExpirationTime = root.expirationTime;
if (rootExpirationTime !== NoWork) {
  requestWork(root, rootExpirationTime);
}
// Flush any sync work that was scheduled by effects
if (!isBatchingUpdates && !isRendering) {
  performSyncWork();
}

function commitRoot (root: FiberRoot, finishedWork: Fiber): void {
  isWorking = true;
  isCommitting = true;
  startCommitTimer();

  invariant(
    root.current !== finishedWork,
    'Cannot commit the same tree as before. This is probably a bug ' +
    'related to the return field. This error is likely caused by a bug ' +
    'in React. Please file an issue.',
  );
  const committedExpirationTime = root.pendingCommitExpirationTime;
  invariant(
    committedExpirationTime !== NoWork,
    'Cannot commit an incomplete root. This error is likely caused by a ' +
    'bug in React. Please file an issue.',
  );
  root.pendingCommitExpirationTime = NoWork;

  // Update the pending priority levels to account for the work that we are
  // about to commit. This needs to happen before calling the lifecycles, since
  // they may schedule additional updates.
  const updateExpirationTimeBeforeCommit = finishedWork.expirationTime;
  const childExpirationTimeBeforeCommit = finishedWork.childExpirationTime;
  const earliestRemainingTimeBeforeCommit =
    childExpirationTimeBeforeCommit > updateExpirationTimeBeforeCommit
      ? childExpirationTimeBeforeCommit
      : updateExpirationTimeBeforeCommit;
  markCommittedPriorityLevels(root, earliestRemainingTimeBeforeCommit);

  let prevInteractions: Set<Interaction> = (null: any);
  if (enableSchedulerTracing) {
    // Restore any pending interactions at this point,
    // So that cascading work triggered during the render phase will be accounted for.
    prevInteractions = __interactionsRef.current;
    __interactionsRef.current = root.memoizedInteractions;
  }

  // Reset this to null before calling lifecycles
  ReactCurrentOwner.current = null;

  let firstEffect;
  if (finishedWork.effectTag > PerformedWork) {
    // A fiber's effect list consists only of its children, not itself. So if
    // the root has an effect, we need to add it to the end of the list. The
    // resulting list is the set that would belong to the root's parent, if
    // it had one; that is, all the effects in the tree including the root.
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
```

## hooks

```typescript

export function scheduleWorkToRoot(fiberRoot?: IRoot) {
    // 第二次的更新
    if (currentRoot && currentRoot.alternate) {
        workInProgressRoot = currentRoot.alternate;
        workInProgressRoot.alternate = currentRoot;
        if (fiberRoot) workInProgressRoot.props = fiberRoot.props;

        // 第一次更新
    } else if (currentRoot) {
        if (fiberRoot) {
            fiberRoot.alternate = currentRoot;
            workInProgressRoot = fiberRoot;
        } else {
            workInProgressRoot = {
                ...currentRoot,
                alternate: currentRoot,
            }
        }
    } else {
        workInProgressRoot = fiberRoot;
    }
    if (workInProgressRoot) {
        workInProgressRoot.firstEffect = workInProgressRoot.lastEffect = workInProgressRoot.nextEffect = null;
    }
    nextUnitOfWork = fiberRoot;
}
```

----

以下为新版本的一些更新（from kasong blog）

![DZQFUlCU0AAkFI5](https://technologybook.tech/assets/img/1.jpeg)

![DZQFWC2VAAEHUln](https://technologybook.tech/assets/img/2.jpeg)

> 在`processUpdateQueue`方法中，`shared.pending`环状链表会被剪开并拼接在`baseUpdate`后面。(from kasong blog)
>
> 优先级高的会在updateQueue.shared.pending 中形成环，
>
> 第一阶段 按链表的顺序
>
> 第二阶段 按优先级

###     