## 深入 React Hook

## Reconciliation --深入Fiber
demo示例：
```
class ClickCounter extends React.Component {
    constructor(props) {
        super(props);
        this.state = {count: 0};
        this.handleClick = this.handleClick.bind(this);
    }

    handleClick() {
        this.setState((state) => {
            return {count: state.count + 1};
        });
    }

    componentDidUpdate() {}

    render() {
        return [
            <button key="1" onClick={this.handleClick}>Update counter</button>,
            <span key="2">{this.state.count}</span>
        ]
    }
}
```

### 以上Reconciliation过程分为：
* 更新state上的count；
* 检索并比较 ClickCounter 的children和props；
* 更新span元素的count；

此外还有生命周期钩子函数的调用和更新refs；

以上过程被统称为 Work；该过程依赖不同类型的element执行不同work；

### Fiber使用的数据结构
From React Elements to Fiber nodes

* JSX语法最终调用 React.createElement 创建 Element；

ClickCounter 最终返回数据结构为：
```
[
    {
        $$typeof: Symbol(react.element),
        type: 'button',
        key: "1",
        props: {
            children: 'Update counter',
            onClick: () => { ... }
        }
    },
    {
        $$typeof: Symbol(react.element),
        type: 'span',
        key: "2",
        props: {
            children: 0
        }
    }
]
```
* 所有从render中返回的element被组织成在fiber tree的节点上进行统一管理；fiber tree node在每次render时候不会像element被重新创建，是管理组件状态和DOM的可变数据结构；
> You can think of a fiber as a data structure that represents some work to do or, in other words, a unit of work. Fiber’s architecture also provides a convenient way to track, schedule, pause and abort the work.

* [createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414)用于基于React Element转换成fiber node；更新过程只是更新node数据；还可能根据key移动位置或者删除node；

* ClickCounter 相应fiber tree用图表示为：
[Image linked list](https://cdn-images-1.medium.com/max/800/1*cLqBZRht7RgR9enHet_0fQ.png)

#### current and workInProgress trees
* 第一次render创建出**current**，第一次更新时创建**workInProgress**（表征将来状态）；

所有work都在workInProgress tree上进行，根据current tree上node和更新时render的返回值在workInProgress tree上更新；最终workInProgress tree被commit后将变为current tree，两者互换；

workInProgress 作为草案/备份，一次性更新Dom，保证 **一致性** 原则；

current tree node包含alternate属性指向其workInProgress tree中对应node，反之亦然；

#### Side-effects
副作用包含修改dom和生命周期钩子函数等；fiber node 的effectTag字段用于存储自身相关effect用于commit阶段；

dom元素 effect包含：增删改；class element包含：componentDidMount，componentDidUpdate；
> You’ve likely performed data fetching, subscriptions, or manually changing the DOM from React components before. We call these operations “side effects” (or “effects” for short) because they can affect other components and can’t be done during rendering.

#### Effects list
render阶段结果集，由所有具有Side-effects的node组成的链表；有tree node的nextEffect字段存储；
[Image Effects list](https://cdn-images-1.medium.com/max/800/1*Q0pCNcK1FfCttek32X_l7A.png)
最终使用结果图示为：
[Image children2parent](https://cdn-images-1.medium.com/max/800/1*mbeZ1EsfMsLUk-9hOYyozw.png)

#### Root of the fiber tree
访问fiber tree root使用以下方法：
```
const domContainer = document.querySelector('#container');
ReactDOM.render(React.createElement(ClickCounter), domContainer);

const fiberRoot = query('#container')._reactRootContainer._internalRoot
```

#### Fiber node structure
```
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    alternate: null,
    key: null,
    updateQueue: null,
    memoizedState: {count: 0},
    pendingProps: {},
    memoizedProps: {},
    tag: 1, // 用于 createFiberFromTypeAndProps
    effectTag: 0,
    nextEffect: null
}

{
    stateNode: new HTMLSpanElement,
    type: "span",
    alternate: null,
    key: "2",
    updateQueue: null,
    memoizedState: null,
    pendingProps: {children: 0},
    memoizedProps: {children: 0},
    tag: 5,
    effectTag: 0,
    nextEffect: null
}
```

### 算法实现
一次Reconciliation过程分为两个阶段：render和commit；
* render阶段依赖setState和render构建或者更新tree，并最终生成side-effect-list；commit阶段执行收集的副作用，更新UI；
* render 不涉及 dom更新，可支持异步渲染和即将支持多线程渲染；commit则是同步渲染，保证数据一致性；

#### 1.Render
* 总是从HostRoot开始，调用renderRoot；但是会调过已经处理过的节点，直到遇到存在未处理work node；
```
function workLoop(isYieldy) {
  if (!isYieldy) {
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {...}
}
```
* 遍历涉及4个方法：
```
   performUnitOfWork
   beginWork
   completeUnitOfWork
   completeWork
```
[Image Render](https://cdn-images-1.medium.com/max/800/1*A3-yF-3Xf47nPamFpRm64w.gif)
* performUnitOfWork 遍历child
* completeUnitOfWork 遍历sibling，parent；
* beginWork, completeWork执行相应work；

#### 2.Commit
* 从 `completeRoot` 开始，此时有两个tree和两个effects list；
* 更新过程主要函数 commitRoot执行以下步骤：
```
   function commitRoot(root, finishedWork) {
       commitBeforeMutationLifecycles()
       commitAllHostEffects();
       root.current = finishedWork;
       commitAllLifeCycles();
   }

   1.getSnapshotBeforeUpdate, 前提有Snapshot effect；
   2.componentWillUnmount，Deletion effect
   3.dom 增删改
   4.finishedWork 和 current 互换；
   5.componentDidMount，Placement effect
   6.componentDidUpdate，Update effect
```

* Pre-mutation
class即为getSnapshotBeforeUpdate；
```
function commitBeforeMutationLifecycles() {
    while (nextEffect !== null) {
        const effectTag = nextEffect.effectTag;
        if (effectTag & Snapshot) {
            const current = nextEffect.alternate;
            commitBeforeMutationLifeCycles(current, nextEffect);
        }
        nextEffect = nextEffect.nextEffect;
    }
}
```
* DOM updates
```
function commitAllHostEffects() {
    switch (primaryEffectTag) {
        case Placement: {
            commitPlacement(nextEffect);
            ...
        }
        case PlacementAndUpdate: {
            commitPlacement(nextEffect);
            commitWork(current, nextEffect);
            ...
        }
        case Update: {
            commitWork(current, nextEffect);
            ...
        }
        case Deletion: {
            commitDeletion(nextEffect);
            ...
        }
    }
}
```
* Post-mutation
commitAllLifecycles 调用componentDidUpdate and componentDidMount；

---------------------------------------------------------------------------------------------

## 详解ClickCounter更新过程
### Render phases
* 更新state的count；
* 调用ClickCounter render返回children；
* 更新span的count props；

### Commit phases
* 更新span textContent；
* 调用componentDidUpdate；

### Scheduling updates
每一类React Element都有对应的updater，其作用是桥接React Core和component，ReactDOM, React Native，SSR，test 都有不同实现；对于ReactDOM，class对应classComponentUpdater；classComponentUpdater 用于维护更新队列和组织work；
```
// 添加到updateQueue后的node
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    updateQueue: {
         baseState: {count: 0}
         firstUpdate: {
             next: {
                 payload: (state) => { return {count: state.count + 1} }
             }
         },
         ...
     },
     ...
}
```
### Render
* 更新过程从调用renderRoot开始，从HostRoot遍历tree；
* 所有work在workInProgress tree上进行，需要先clone alternate对应node，如果没有（第一次update）则需要调用 createWorkInProgress 创建；
* beginWork 可打断点debug node type
> Since this function is executed for every Fiber node in a tree it’s a good place to put a breakpoint if you want to debug the render phase. I do that often and check the type of a Fiber node to pin down the one I need.
```
function beginWork(current$$1, workInProgress, ...) {
    ...
    switch (workInProgress.tag) {
        ...
        case FunctionalComponent: {...}
        case ClassComponent:
        {
            ...
            return updateClassComponent(current$$1, workInProgress, ...);
        }
        case HostComponent: {...}
        case ...
}
```
根据tag，执行相应的case：updateClassComponent；新建或者重用；
```
function updateClassComponent(current, workInProgress, Component, ...) {
    ...
    const instance = workInProgress.stateNode;
    let shouldUpdate;
    if (instance === null) {
        ...
        // In the initial pass we might need to construct the instance.
        constructClassInstance(workInProgress, Component, ...);
        mountClassInstance(workInProgress, Component, ...);
        shouldUpdate = true;
    } else if (current === null) {
        // In a resume, we'll already have an instance we can reuse.
        shouldUpdate = resumeMountClassInstance(workInProgress, Component, ...);
    } else {
        shouldUpdate = updateClassInstance(current, workInProgress, ...);
    }
    return finishClassComponent(current, workInProgress, Component, shouldUpdate, ...);
}
```
* Processing updates
```
   1.执行updateQueue，生成new state；
   2.getDerivedStateFromProps
   3.shouldComponentUpdate 返回false则不执行 render；
   4.add an effect to trigger componentDidUpdate lifecycle hook；
   5.更新state和props；
   function updateClassInstance(current, workInProgress, ctor, newProps, ...) {
       const instance = workInProgress.stateNode;

       const oldProps = workInProgress.memoizedProps;
       instance.props = oldProps;
       if (oldProps !== newProps) {
           callComponentWillReceiveProps(workInProgress, instance, newProps, ...);
       }

       let updateQueue = workInProgress.updateQueue;
       if (updateQueue !== null) {
           processUpdateQueue(workInProgress, updateQueue, ...);
           newState = workInProgress.memoizedState;
       }

       applyDerivedStateFromProps(workInProgress, ...);
       newState = workInProgress.memoizedState;

       const shouldUpdate = checkShouldComponentUpdate(workInProgress, ctor, ...);
       if (shouldUpdate) {
           instance.componentWillUpdate(newProps, newState, nextContext);
           workInProgress.effectTag |= Update;
           workInProgress.effectTag |= Snapshot;
       }

       instance.props = newProps;
       instance.state = newState;

       return shouldUpdate;
   }
```
* Render 结束后的 ClickCounter
```
{
    effectTag: 0,
    elementType: class ClickCounter,
    firstEffect: null,
    memoizedState: {count: 0},
    type: class ClickCounter,
    stateNode: {
        state: {count: 0}
    },
    updateQueue: {
        baseState: {count: 0},
        firstUpdate: {
            next: {
                payload: (state, props) => {…}
            }
        },
        ...
    }
}
{
    effectTag: 4, // Update
    elementType: class ClickCounter,
    firstEffect: null,
    memoizedState: {count: 1},
    type: class ClickCounter,
    stateNode: {
        state: {count: 1}
    },
    updateQueue: {
        baseState: {count: 1},
        firstUpdate: null,
        ...
    }
}
```
* Reconciling children --finishClassComponent 执行render；
该过程1.创建或更新render返回的element；finishClassComponent返回第一个child；2.更新children的props;

finishClassComponent 调用 reconcileChildren；

```
// fiber node
{
    stateNode: new HTMLSpanElement,
    type: "span",
    key: "2",
    memoizedProps: {children: 0},
    pendingProps: {children: 0},
    ...
}
// return element
{
    $$typeof: Symbol(react.element)
    key: "2"
    props: {children: 1}
    ref: null
    type: "span"
}
```
createWorkInProgress调用生成的新fiber node
```

{
    stateNode: new HTMLSpanElement,
    type: "span",
    key: "2",
    memoizedProps: {children: 0},
    pendingProps: {children: 1},
    ...
}

```
* updates for the Span
外层循环nextUnitOfWork指向span，首先还是执行到 beginWork；
```
function beginWork(current$$1, workInProgress, ...) {
    ...
    switch (workInProgress.tag) {
        case FunctionalComponent: {...}
        case ClassComponent: {...}
        case HostComponent:
          return updateHostComponent(current, workInProgress, ...);
        case ...
}
// 最后更新pendingProps 到memoizedProps；
function performUnitOfWork(workInProgress) {
    ...
    next = beginWork(current$$1, workInProgress, nextRenderExpirationTime);
    workInProgress.memoizedProps = workInProgress.pendingProps;
    ...
}
```
```
   1.准备dom更新
   2.给span fiber更新updateQueue；
   3.更新effect；
function completeWork(current, workInProgress, ...) {
    ...
    switch (workInProgress.tag) {
        case FunctionComponent: {...}
        case ClassComponent: {...}
        case HostComponent: {
            ...
            updateHostComponent(current, workInProgress, ...);
        }
        case ...
    }
}
```
更新前后span fiber 对比：
```
{
    stateNode: new HTMLSpanElement,
    type: "span",
    effectTag: 0
    updateQueue: null
    ...
}
{
    stateNode: new HTMLSpanElement,
    type: "span",
    effectTag: 4,
    updateQueue: ["children", "1"],
    ...
}

```
* Effects list
compliteUnitOfWork最终生成list，如图：
[Image Effects List](https://cdn-images-1.medium.com/max/800/1*TRmFSeuOuWlY3HXh86cvDA.png)
等价于：
[Image List](https://cdn-images-1.medium.com/max/800/1*ekyRg-8mqawxWoTpcPvM1w.png)

### Commit
* 入口 `completeRoot`；root.finishedWork = null;
```
// 结果集：
{ type: ClickCounter, effectTag: 5 } // 末位为1表示render阶段所有work完成？
{ type: 'span', effectTag: 4 }
```
* Apply effects
每个方法遍历effects list；
```
function commitRoot(root, finishedWork) {
    commitBeforeMutationLifecycles(); // getSnapshotBeforeUpdate
    commitAllHostEffects(); // dom 更新；
    root.current = finishedWork;
    commitAllLifeCycles();
}

// 更新span
function updateHostEffects() {
    switch (primaryEffectTag) {
      case Placement: {...}
      case PlacementAndUpdate: {...}
      case Update:
        {
          var current = nextEffect.alternate;
          commitWork(current, nextEffect);
          break;
        }
      case Deletion: {...}
    }
}
function updateDOMProperties(domElement, updatePayload, ...) {
  for (let i = 0; i < updatePayload.length; i += 2) {
    const propKey = updatePayload[i];
    const propValue = updatePayload[i + 1];
    if (propKey === STYLE) { ...}
    else if (propKey === DANGEROUSLY_SET_INNER_HTML) {...}
    else if (propKey === CHILDREN) {
      setTextContent(domElement, propValue);
    } else {...}
  }
}

// 调用 componentDidUpdate
function commitAllLifeCycles(finishedRoot, ...) {
    while (nextEffect !== null) {
        const effectTag = nextEffect.effectTag;

        if (effectTag & (Update | Callback)) {
            const current = nextEffect.alternate;
            commitLifeCycles(finishedRoot, current, nextEffect, ...);
        }

        if (effectTag & Ref) {
            commitAttachRef(nextEffect);
        }

        nextEffect = nextEffect.nextEffect;
    }
}

function commitLifeCycles(finishedRoot, current, ...) {
  ...
  switch (finishedWork.tag) {
    case FunctionComponent: {...}
    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (finishedWork.effectTag & Update) {
        if (current === null) {
          instance.componentDidMount();
        } else {
          ...
          instance.componentDidUpdate(prevProps, prevState, ...);
        }
      }
    }
    case HostComponent: {...}
    case ...
}
```

---------------------------------------------------------------------------------------------

## 详解ClickCounter Hook实现的更新过程
```
export default function ExampleHooks() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
Mounting 阶段先不详谈；

### render
* 点击button后，触发setCount，实现则会调用 dispatchAction；新值action会被更新到hook的queue上；
```
function dispatchAction(fiber, queue, action) {
  ...
  var _update2 = {
    expirationTime: _expirationTime,
    action: action,
    eagerReducer: null,
    eagerState: null,
    next: null
  }; // Append the update to the end of the list.

  queue.last = _update2;
}
```
* renderRoot 从HostRoot 开始遍历所有fiber node；如上，根据其tag进入 updateFunctionComponent，并进入renderWithHooks；
```
function renderWithHooks(current, workInProgress, Component, props, refOrContext, nextRenderExpirationTime)
{
    ...
    // Component 既是function component，本例为 ExampleHooks；Component执行时调用useState；
    var children = Component(props, refOrContext);
}

const HooksDispatcherOnMount: Dispatcher = {
  useState: mountState,
};

// update 过程调用updateState
const HooksDispatcherOnUpdate: Dispatcher = {
  useState: updateState, // call updateReducer
};

```
updateReducer 负责执行 hook queue中的更新队列；new state会被更新在hook.memoizedState；
```
// first 为queue中首个更新项；然后通过next遍历执行所有action执行对应的reducer；
if (first !== null) {
  var _newState = baseState;
  var _update = first;

  do {
    var _action2 = _update.action;
    _newState = reducer(_newState, _action2);

    prevUpdate = _update;
    _update = _update.next;
  } while (_update !== null && _update !== first);

  hook.memoizedState = _newState;
}

var dispatch = queue.dispatch;
return [hook.memoizedState, dispatch];
```
* renderWithHooks 返回 children，传入 reconcileChildren，内部根据element 构建fiber node；然后进入child继续外层循环；

### commit
completeRoot 起始点打断点查看finishedWork；
* hostroot firstEffect -> p;
* effectTag：ExampleHooks 1;    p 4 nextEffect -> button;   button 4 nextEffect -> null;
* sideEffect list:  hostroot -> p -> button -> null;
```
scheduler.unstable_runWithPriority(scheduler.unstable_ImmediatePriority, function () {
  commitRoot(root, finishedWork);
});
```
执行 commitRoot，和上例类似过程；最终改变p的value；



[Inside Fiber](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e)