# React Fiber源码理解


## 前言
我们知道，React Fiber是React v16中新的reconciliation引擎，是React团队用时2年对Stack Reconciler版本的核心算法进行的重写。它的主要目标是实现虚拟DOM的增量渲染，可以将渲染工作拆分成块并将其分散到多个帧的能力。在新的更新到来时，可以暂停、中止或复用工作的能力，可以为不同类型的更新分配优先级顺序的能力。这是React设计的一种体现，相比于业界一些流行库在计算更新时采用的“push”方法，React坚持“pull”方法从而延迟到必要时进行计算。这听起来很有意思。

理解React的核心架构机制对我们更好理解React思想以及后续版本新特性，比如并行渲染等都会有很大帮助，本文基于React 16.8.6版本源码，输出一些我的理解，希望对你也有帮助，如有不对，还望指正。

## 前提概念
在深入理解React fiber架构的实现机制之前，有必要先把几个核心的基本概念抛出来，这些基本概念对理解整个机制运行非常有帮助，值得反复咀嚼。一开始不理解这些基本概念没有关系，在逐步地了解和深入整个fiber机制过程中思考，相信可以慢慢领悟到react提出这些概念的原理。

### Fiber 对象
> 文件位置：packages/react-reconciler/src/ReactFiber.js


一个fiber对象是表征work的一个基本单元。

每一个React元素对应一个fiber对象，fibers是一个基于child, sibling 和 return属性构成的链表。
fiber对象核心的属性和含义如下所示：

```javascript
Fiber = {
  // 标识fiber类型的标签，详情参看下述WorkTag
  tag: WorkTag,
  
  // The local state associated with this fiber.参看下述stateNode
  stateNode: any,

  // The Fiber to return to after finishing processing this one.
  // This is effectively the parent, but there can be multiple parents (two)
  // so this is only the parent of the thing we're currently processing.
  // 指向父节点
  return: Fiber | null,

  // Singly Linked List Tree Structure.
  // 指向子节点
  child: Fiber | null,
  // 指向兄弟节点
  sibling: Fiber | null,

  // Input is the data coming into process this fiber. Arguments. Props.
  // This type will be more specific once we overload the tag.
	// 在开始执行时设置props参数，
  pendingProps: any, 
  // The props used to create the output.
	// 在结束时设置的props
  memoizedProps: any, 

  // A queue of state updates and callbacks.
  updateQueue: UpdateQueue<any> | null,

  // The state used to create the output
  // 当前 state
  memoizedState: any,
    
  // Effect
  // 详情查看以下 effectTag
  effectTag: SideEffectTag,

  // 详情查看以下 effectList
  // Singly linked list fast path to the next fiber with side-effects.
  nextEffect: Fiber | null,
  // The first and last fiber with side-effect within this subtree. This allows
  // us to reuse a slice of the linked list when we reuse the work done within
  // this fiber.
  firstEffect: Fiber | null,
  lastEffect: Fiber | null,

  // Represents a time in the future by which this work should be completed.
  expirationTime: ExpirationTime,

  // This is used to quickly determine if a subtree has no pending changes.
  childExpirationTime: ExpirationTime,

  // This is a pooled version of a Fiber. Every fiber that gets updated will
  // eventually have a pair. There are cases when we can clean up pairs to save
  // memory if we need to.
  alternate: Fiber | null,
}
```

从React元素创建一个fiber对象
> 文件位置：react-reconciler/src/ReactFiber.js


```javascript
export function createFiberFromElement(
  element: ReactElement,
  mode: TypeOfMode,
  expirationTime: ExpirationTime,
): Fiber {
  const fiber = createFiberFromTypeAndProps(
    type,
    key,
    pendingProps,
    owner,
    mode,
    expirationTime,
  );
  return fiber;
}
```


### child、silbing、return
fiber对象的属性，这些属性指向其他fiber，表征当前工作单元的下一个工作单元，用于描述fiber的递归树结构。

child： 对应于父fiber节点的子fiber
silbing： 对应于fiber节点的同类兄弟节点
return： 对应于fiber节点的父节点

相对于React v16之前的版本，正是得益于fiber对象的child、sibing和return属性构成的单链表结构以及fiber对象中存储的上下文信息，才使得scheduler可以达到暂停、中止、重新开始等并发模式的新特性。

### work
在React reconciliation过程中出现的各种比如state update，props update 或 refs update等必须执行计算的活动，这些活动我们在Fiber架构体系里面统一称之为 “work”。

### workTag
> 文件位置：shared/ReactWorkTags.js

workTag 类型，用于描述一个React元素的类型，即为上述fiber对象的 fiber.tag，如下显示：

```javascript
export const FunctionComponent = 0;
export const ClassComponent = 1;
export const IndeterminateComponent = 2; // Before we know whether it is function or class
export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
export const HostPortal = 4; // A subtree. Could be an entry point to a different renderer.
export const HostComponent = 5;
export const HostText = 6;
export const Fragment = 7;
export const Mode = 8;
export const ContextConsumer = 9;
export const ContextProvider = 10;
export const ForwardRef = 11;
export const Profiler = 12;
export const SuspenseComponent = 13;
export const MemoComponent = 14;
export const SimpleMemoComponent = 15;
export const LazyComponent = 16;
export const IncompleteClassComponent = 17;
export const DehydratedSuspenseComponent = 18;
export const EventComponent = 19;
export const EventTarget = 20;
export const SuspenseListComponent = 21;
```

### stateNode
一个组件、一个DOM节点或其他跟fiber节点相关联的React元素的实例的引用。通常，我们可以说这个属性是用于保存与一个fiber相关联的本地状态。即上述fiber对象的 fiber.stateNode。

### effectTag
> 文件位置：shared/ReactSideEffectTags.js


我们通常把手动更改DOM或在生命周期中执行数据请求、订阅等操作，不能在render阶段完成的work，我们称这些工作为side effects。除了update等常见的effect以外，fiber节点提供了一种方便的机制去追踪effect。每一个fiber节点都有一个和它相关联的effct，这些被编码为effectTag字段对应的值，即为fiber.effectTag，如下所示：

```javascript
// Don't change these two values. They're used by React Dev Tools.
export const NoEffect = /*              */ 0b000000000000;
export const PerformedWork = /*         */ 0b000000000001;

// You can change the rest (and add more).
export const Placement = /*             */ 0b000000000010;
export const Update = /*                */ 0b000000000100;
export const PlacementAndUpdate = /*    */ 0b000000000110;
export const Deletion = /*              */ 0b000000001000;
export const ContentReset = /*          */ 0b000000010000;
export const Callback = /*              */ 0b000000100000;
export const DidCapture = /*            */ 0b000001000000;
export const Ref = /*                   */ 0b000010000000;
export const Snapshot = /*              */ 0b000100000000;
export const Passive = /*               */ 0b001000000000;

// Passive & Update & Callback & Ref & Snapshot
export const LifecycleEffectMask = /*   */ 0b001110100100;

// Union of all host effects
export const HostEffectMask = /*        */ 0b001111111111;

export const Incomplete = /*            */ 0b010000000000;
export const ShouldCapture = /*         */ 0b100000000000;
```

### Priority
> 文件位置：react-reconciler/src/SchedulerWithReactIntegration.js


在fiber体系中，一些表示work优先级的数字，如下列出了不同的优先级级别及其表示的内容：

```javascript
// We use ascending numbers so we can compare them like numbers. 
export const ImmediatePriority: ReactPriorityLevel = 99;
export const UserBlockingPriority: ReactPriorityLevel = 98;
export const NormalPriority: ReactPriorityLevel = 97;
export const LowPriority: ReactPriorityLevel = 96;
export const IdlePriority: ReactPriorityLevel = 95;
```

### Reconciliation和Scheduling
reconciliation：React通过diff算法来比较vdom，用于确认哪些部分需要更改。
scheduling_：_用于确定work何时执行的过程。

### render阶段和commit阶段
这是React团队Dan Abramov 画的一张生命周期阶段图，他把React的生命周期主要分为两个阶段：Render阶段和Commit阶段。其中Commit阶段又可以细分为Pre-commit阶段和Commit阶段，如下图所示：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/205608/1564662415011-aa685991-8be8-44ca-8bed-95c4dd579e70.png#align=left&display=inline&height=383&name=image.png&originHeight=766&originWidth=1400&size=361503&status=done&width=700)

在render阶段中，理解work可以异步执行是非常重要的。React可以根据可用时间处理一个或多个fiber节点，然后停止并保存已完成的工作，让出时间去处理一些事件。随后，它从停止的地方继续。不过，有时可能会丢弃已经完成的工作，重新从顶部开始。由于在此阶段执行的工作不会导致任何用户可见的更改(如DOM更新)，所以这些暂停是可能的。相反，在commit阶段，work执行总是同步的。这是因为在此阶段执行的工作将导致用户可见的更改，例如DOM更新。这就是为什么React需要一次性commit并完成这些工作的原因。

从16.3版本开始，在render阶段，以下几个生命周期被认为是不安全，它们将在未来的版本中被移除，上述生命周期图例未被包括进去，如下列表：

- [UNSAFE_]componentWillMount (deprecated)
- [UNSAFE_]componentWillReceiveProps (deprecated)
- [UNSAFE_]componentWillUpdate (deprecated)

这些被标记为不安全的生命周期常常被错误理解甚至被滥用，一些开发人员倾向于将带有副作用的代码放在这些方法中，这在新的异步render方法下可能会导致出问题。在即将到来的Concurrent 模式下，它们仍然很有可能会引发问题。

### current树和workInProgress树
首次渲染后，React生成一个用于渲染UI并能映射应用状态的fiber树，我们通常称之为current树。当React遍历current树，它为每一个存在的fiber节点创建一个alternate属性的替代节点，该节点构成workInProgress树。

所有发生update的work都在workInProgress树中执行，如果alternate属性还未创建，React将在处理update之前在createWorkInProgress函数中创建一个current树的副本，即形成workInProgress树，用于映射新的状态并在commit阶段刷新到屏幕。

### Effects list
effect list列表是基于fiber对象由 firstEffect、nextEffect和lastEffect 属性构成的链表结构。如下图所示：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/205608/1565007636075-981b3e07-dd64-44e1-944b-b92443a8f120.png#align=left&display=inline&height=59&name=image.png&originHeight=103&originWidth=712&size=6229&status=done&width=410)


当render阶段结束，会生成一个effect list。在commit阶段，React循环遍历effect-list并检查该fiber的effect的类型，当发现和他类型相关用途函数，就应用调用，比如DOM改变、相关生命周期函数调用等。

effect list构建的核心代码逻辑如下：
> 文件位置： react-reconciler/src/ReactFiberWorkLoop.js


```javascript
function completeUnitOfWork(unitOfWork: Fiber): Fiber | null {
	// Append all the effects of the subtree and this fiber onto the effect
  // list of the parent. The completion order of the children affects the
  // side-effect order.
  if (returnFiber.firstEffect === null) {
    returnFiber.firstEffect = workInProgress.firstEffect;
  }
  if (workInProgress.lastEffect !== null) {
    if (returnFiber.lastEffect !== null) {
      returnFiber.lastEffect.nextEffect = workInProgress.firstEffect;
    }
    returnFiber.lastEffect = workInProgress.lastEffect;
  }
  
  if (returnFiber.lastEffect !== null) {
    returnFiber.lastEffect.nextEffect = workInProgress;
  } else {
    returnFiber.firstEffect = workInProgress;
  }
  returnFiber.lastEffect = workInProgress;
}
```

整个构建过程可以大致归结为如下：
1）遍历到workInProgress1节点时，当returnFiber.firstEffect和returnFiber.lastEffect都为null时，则returnFiber.firstEffect=workInProgress1和returnFiber.lastEffect=workInProgress1。
2）此时继续查找，当找到workInProgress1的兄弟节点workInProgress2时，执行returnFiber.lastEffect.nextEffect = workInProgress2，目的是将workInProgress1.nextEffect = workInProgress2。同时，returnFiber.lastEffect = workInProgress2，即更新returnFiber的lastEffect到workInProgress2。
3）此时，形成 workInProgress1.nextEffect = workInProgress2，returnFiber.firstEffect = workInProgress1，returnFiber.lastEffect = workInProgress2，继续查找同类节点和父节点，以此类推。

详情部分可以查看文章底部参考《effect-list fiber链表构建过程》部分，该视频详细展示了effect-list构建流程。


## render阶段
在本文中，我们主要着重于在ReactDOM中的updater对象的实现，以类组件为例，它是一个classComponentUpdater，用于获取fiber实例、update队列和调度 work。我们假设已经开始调用了一个setState方法。

### enqueueSetState
每个React组件都有一个相关联的updater，作为组件和React核心库之间的桥梁。这使得setState可以通过ReactDOM、React Native、服务器端渲染和测试工具等不同的方式来实现。

Component函数的原型对象上挂载了 setState 方法，调用组件实例对象 this.updater.enqueueSetState 

> 文件位置：react/src/ReactBaseClasses.js


```javascript
// Component函数
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

// Component原型对象挂载setState
Component.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState')
}
```

 enqueueSetState 方法如下：
> 文件位置：react-reconciler/src/ReactFiberClassComponent.js


enqueueUpdate(fiber, update)：插入update到队列 
scheduleWork(fiber, expirationTime)： 调度work

```javascript
const classComponentUpdater = {
  /*
   inst：触发setState的class组件实例
   payload: setState的参数对象
  */
  enqueueSetState(inst, payload, callback) {
    // 获取 inst._reactInternalFiber 的 fiber 对象
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTime();

    // 计算到期时间
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update = createUpdate(expirationTime, suspenseConfig);
    update.payload = payload;

    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  }
}
```

### enqueueUpdate
> 文件位置：react-reconciler/src/ReactUpdateQueue.js


fiber.updateQueue是一个具有updates优先级的链表（UpdateQueue is a linked list of prioritized updates）

跟Fiber一样，update 队列也是成对出现：一个代表屏幕可见状态的 current 队列，一个在commit阶段之前可被异步计算和处理的work-in-progress 队列。如果一个work-in-progress队列在完成之前被丢弃，则将会通过克隆一个curent队列来创建一个新的work-in-progress队列。

核心代码逻辑如下：
```javascript
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // Update queues are created lazily.
  const alternate = fiber.alternate;
  let queue1;
  let queue2;
  if (alternate === null) {
    // There's only one fiber.
    queue1 = fiber.updateQueue;
    queue2 = null;
    if (queue1 === null) {
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // There are two owners.
    queue1 = fiber.updateQueue;
    queue2 = alternate.updateQueue;
    if (queue1 === null) {
      if (queue2 === null) {
        // Neither fiber has an update queue. Create new ones.
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState,
        );
      } else {
        // Only one fiber has an update queue. Clone to create a new one.
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {
        // Only one fiber has an update queue. Clone to create a new one.
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }
  if (queue2 === null || queue1 === queue2) {
    // There's only a single queue.
    appendUpdateToQueue(queue1, update);
  } else {
    // There are two queues. We need to append the update to both queues,
    // while accounting for the persistent structure of the list — we don't
    // want the same update to be added multiple times.
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // One of the queues is not empty. We must add the update to both queues.
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      // Both queues are non-empty. The last update is the same in both lists,
      // because of structural sharing. So, only append to one of the lists.
      appendUpdateToQueue(queue1, update);
      // But we still need to update the `lastUpdate` pointer of queue2.
      queue2.lastUpdate = update;
    }
  }
}

/*
	1）如果queue.lastUpdate为null，
     则将queue的firstUpdate和lastUpdate分别指向最新update
  2）如果存在queue.lastUpdate，
     则将上一次lastUpate通过next属性指向最新update，把lastUpdate指向最新update
*/
function appendUpdateToQueue<State>(
  queue: UpdateQueue<State>,
  update: Update<State>,
) {
  // Append the update to the end of the list.
  if (queue.lastUpdate === null) {
    // Queue is empty
    queue.firstUpdate = queue.lastUpdate = update;
  } else {
    queue.lastUpdate.next = update;
    queue.lastUpdate = update;
  }
}
```

两个队列都是一个可持久化的链表结构。为了调度update，我们插入它到每一个队列的末尾。在持久化链表中，每一个队列都有一个指向还未处理的first update的指针。由于总是在work-in-progress 上工作，它的指针位置总是等于或大于current 队列的指针位置。current update的指针只有在commit 阶段被work-in-progress替换更新。

举例：
```javascript
Current pointer:           A - B - C - D - E - F
Work-in-progress pointer:              D - E - F
                                       ^
如上箭头所示：相比current，Work-in-progress处理updates更多
```

增加两个队列的主要原因是为了避免丢失 updates。比如，如果我们只增加update到work-in-progress 队列，当work-in-progress通过从current队列中克隆而重新开始时，一些updates可能会被丢失。同样的，如果我们只增加updates到 current 队列，当一个已存在的work-in-progress队列commit并替换current队列，updates也会被丢失。通过在两个队列都增加updates，我们可以保证updates始终为下一个work-in-progress的一部分。并且，因为work-in-progress队列一旦 commit将会成为current 队列，并不会出现相同的update被应用两次的风险。

update并非通过优先级来存储，而是通过插入的顺序，最新的update总是被添加到链表的末尾。然而，优先级依然非常重要。当在 render 阶段处理update时，只有那些优先级更高的update能获取到结果。如果是由于优先级不够而忽略的update，它依然保持在队列中等待更低优先级的render阶段被处理。不管优先级怎么样，所有的update都保存在队列中。这意味着高优先级的update在两次不同优先级下有时候会被处理两次。我们同样也保持一个base state的跟踪，base state 代表在队列中 updateQueue.fitstUpdate 被应用之前的状态。

举例：
```javascript
给定一个''的 baseSate 和如下的updates
A1 - B2 - C1 - D2

数字表示优先级。React将通过两次独立的render阶段来处理这些updates：
  首次render, 1级优先级:
  Base state: ''
  Updates: [A1, C1]
  Result state: 'AC'

第二次render, 2级优先级:
Base state: 'A'            <-  The base state does not include C1,
because B2 was skipped.
Updates: [B2, C1, D2]      <-  C1 was rebased on top of B2
Result state: 'ABCD'
```

### renderRoot
> 文件位置：react-reconciler/src/ReactFiberWorkLoop.js


reconciliation 过程总是renderRoot开始，函数调用栈如下所示：
scheduleWork -->  scheduleCallbackForRoot  --> renderRoot

renderRoot主要的代码逻辑如下：

```javascript
function renderRoot(
  root: FiberRoot,
  expirationTime: ExpirationTime,
  isSync: boolean,
) | null {
  do {
    if (isSync) {
      workLoopSync();
    } else {
      workLoop();
    }
  } while (true);
}

/*
 所有的fiber节点都在workLoop 中被处理
*/
function workLoop() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

### performUnitOfWork
函数调用栈如下：
> performUnitOfWork  -->  beginWork -->  updateClassComponent --> finishedComponent --> completeUnitOfWork


reconciliation算法总是从最顶层的HostRootfiber节点开始进行树的遍历。然而，React会跳过已经处理过的fiber节点，直到找到还未完成工作的节点。例如，如果在组件树的深处调用setState, React将从顶部开始，但会快速跳过父节点，直到到达调用了setState方法的组件。

所有的fiber节点都在workLoop处理。

当React遍历fibers树时，它使用workInProgress来知道是否有任何其他fiber节点具有未完成的工作。 处理当前fiber后，workInProgress将包含对树中下一个fiber节点的引用，若为null，则React退出工作循环并准备进行 commit更改。

整个过程采用的是深度优先搜索算法，优先查找workInProgress是否存在子节点，如果存在，继续查找该子节点的子节点，直到搜索的子节点为null，则返回继续查找该子节点的同类兄弟节点。同样，继续查找该兄弟节点的子节点的子节点，以此类推。

详细流程可以点击如下链接查看示例：
[https://stackblitz.com/edit/js-ntqfil?file=index.js](https://stackblitz.com/edit/js-ntqfil?file=index.js)

核心代码如下所示：
> 文件位置：react-reconciler/src/ReactFiberWorkLoop.js


```javascript
function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;

  let next;
  next = beginWork(current, unitOfWork, renderExpirationTime);

  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    next = completeUnitOfWork(unitOfWork);
  }

  return next;
}
```

### beginWork
> 文件位置：react-reconciler/src/ReactFiberBeginWork.js


beginWork函数主要通过 workInProgress.tag 来确定fiber 节点的 work 类型，并返回相应tag对应的函数来执行工作。以ClassComponent组件为例， 检测workInProgress.tag 为ClassComponent，则执行 updateClassComponent、finishClassComponent函数，最后返回 workInProgress.child。

主要代码逻辑如下所示：
```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
	switch (workInProgress.tag) {
  	case FunctionComponent:
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime,
      );
    case ClassComponent:
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime,
      )
    case HostRoot:
      ...
    case HostComponent:
      ...
    case HostText:
    
    ...
  }
}
    
/*
  以 updateClassComponent 为例
  返回 finishClassComponent 的执行结果
*/ 

function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps,
  renderExpirationTime: ExpirationTime,
) {
  const nextUnitOfWork = finishClassComponent(
    current,
    workInProgress,
    Component,
    shouldUpdate,
    hasContext,
    renderExpirationTime,
  );
  return nextUnitOfWork
}
    
/*
	返回 workInProgress的子节点
*/
function finishClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  shouldUpdate: boolean,
  hasContext: boolean,
  renderExpirationTime: ExpirationTime,
) {
  return workInProgress.child;
}
```

以下是按执行顺序在函数中执行的一些重要操作：

- 调用UNSAFE_componentWillReceiveProps()钩子(已废弃)
- 处理在updateQueue中的update，并生成新state状态
- 使用新的state状态调用getDerivedStateFromProps，并获得结果
- 调用shouldComponentUpdate，以确定组件是否update;如果为false，则跳过整个render过程，包括对该组件及其子组件调用render。否则，继续执行update
- 调用UNSAFE_componentWillUpdate(已废弃)
- 添加一个effect来触发componentDidUpdate生命周期钩子（虽然在render阶段添加了调用componentDidUpdate的effect，但该方法将在commit阶段执行）
- 更新组件实例上的state和props

**一旦上述操作完成之后，React进入finishClassComponent。在这里，React调用组件实例上的render方法，并将其diff算法应用于子组件。**
**
注意此时workInProgress节点的effectTag和updateQueue属性值的变化。effectTag值不再是0，而是对应的effect数值。updateQueue的baseState包含了用于effect的有效负载值。这些变化是在接下来的commit 阶段中，React用于对这个节点做需要做的工作。

render阶段完成，现在，它可以将完成的alternate树分配到fiberRoot上的finishedWork属性。这是需要committed到屏幕的新树。它可以在render阶段之后立即处理，也可以在浏览器让出时间之后处理。

如下代码所示：
```javascript
function renderRoot() {
	// We now have a consistent tree. The next step is either to commit it, or, if
  // something suspended, wait to commit it after a timeout.
  stopFinishedWorkLoopTimer();

  root.finishedWork = root.current.alternate;
  root.finishedExpirationTime = expirationTime;
}

```


### completeUnitOfWork
> 文件位置：react-reconciler/src/completeUnitOfWork.js


React在completeUnitOfWork函数中构建effect-list，源码请查看如下构建effect-list部分，详情描述请查看上述Effect list概念部分。

是深度优先搜索算法一部分，获取workInProgress.alternate、父节点workInProgress.return和workInProgress.sibling，如果存在兄弟节点则返回。否则，返回父节点。

主要源码如下：
```javascript
function completeUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  workInProgress = unitOfWork;
  do {
    const current = workInProgress.alternate;
    const returnFiber = workInProgress.return;
		
    /*
    	构建 effect-list部分
    */
    // Append all the effects of the subtree and this fiber onto the effect
    // list of the parent. The completion order of the children affects the
    // side-effect order.
    if (returnFiber.firstEffect === null) {
      returnFiber.firstEffect = workInProgress.firstEffect;
    }
    if (workInProgress.lastEffect !== null) {
      if (returnFiber.lastEffect !== null) {
        returnFiber.lastEffect.nextEffect = workInProgress.firstEffect;
      }
      returnFiber.lastEffect = workInProgress.lastEffect;
    }

    if (returnFiber.lastEffect !== null) {
      returnFiber.lastEffect.nextEffect = workInProgress;
    } else {
      returnFiber.firstEffect = workInProgress;
    }
    returnFiber.lastEffect = workInProgress;
    
    
    /*
    	深度优先搜索算法一部分
    */
    const siblingFiber = workInProgress.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      return siblingFiber;
    }
    // Otherwise, return to the parent
    workInProgress = returnFiber;
  } while (workInProgress !== null);
}
```

## commit 阶段
commit阶段是React更新DOM并调用pre-commit phase和commit phase生命周期方法的地方。为此，它将遍历在上一个render阶段构建的effect list并应用它们。

**注意，与render阶段不同，commit阶段的执行始终是同步的。**

### commitRootImpl
commit阶段被分为几个子阶段。每个子阶段都单独进行effect list传递。所有的mutation effects都会在所有的layout effects之前执行。

被分为如下三个子阶段：

- before mutation
- mutation phase
- layout phase

第一个子阶段是“before mutation”阶段。 在React mutate之前，React使用此阶段读取 host tree的state状态。 这是调用getSnapshotBeforeUpdate生命周期的地方。

下一个子阶段是“mutation phase”，在这个阶段，React 会改变host tree。
当该阶段执行结束时，work-in-progress树会变成current树，这必须发生在“mutation phase”阶段之后，以便于在componentWillUnmount生命周期内，仍然是之前的current树。但是，也要发生在“layout phase”阶段之前，以便于在componentDidMount / Update生命周期间，current树是已完成的work操作的。

下一个子阶段是“layout phase”，在这个阶段host tree已经被更改并调用 effects。componentDidMount / Update等生命周期在这个阶段被执行。

> 文件位置：react-reconciler/src/ReactFiberWorkLoop.js

主要的源码如下：

```javascript
function commitRootImpl(root) {
  // Get the list of effects.
  let firstEffect;
  if (finishedWork.effectTag > PerformedWork) {
    // A fiber's effect list consists only of its children, not itself. So if
    // the root has an effect, we need to add it to the end of the list. The
    // resulting list is the set that would belong to the root's parent, if it
    // had one; that is, all the effects in the tree including the root.
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    // There is no effect on the root.
    firstEffect = finishedWork.firstEffect;
  }

  if (firstEffect !== null) {
    // The commit phase is broken into several sub-phases. We do a separate pass
    // of the effect list for each phase: all mutation effects come before all
    // layout effects, and so on.

    // The first phase a "before mutation" phase. We use this phase to read the
    // state of the host tree right before we mutate it. This is where
    // getSnapshotBeforeUpdate is called.

    do {
      try {
        commitBeforeMutationEffects();
      } catch (error) {
        nextEffect = nextEffect.nextEffect;
      }
    } while (nextEffect !== null);


    // The next phase is the mutation phase, where we mutate the host tree.
    nextEffect = firstEffect;
    do {
      try {
        commitMutationEffects();
      } catch (error) {
        nextEffect = nextEffect.nextEffect;
      }
    } while (nextEffect !== null);
  

    // The work-in-progress tree is now the current tree. This must come after
    // the mutation phase, so that the previous tree is still current during
    // componentWillUnmount, but before the layout phase, so that the finished
    // work is current during componentDidMount/Update.
    root.current = finishedWork;

    // The next phase is the layout phase, where we call effects that read
    // the host tree after it's been mutated. The idiomatic use case for this is
    // layout, but class component lifecycles also fire here for legacy reasons.
    nextEffect = firstEffect;
    do {
      try {
        commitLayoutEffects(root, expirationTime);
      } catch (error) {
        invariant(nextEffect !== null, 'Should be working on an effect.');
        captureCommitPhaseError(nextEffect, error);
        nextEffect = nextEffect.nextEffect;
      }
    } while (nextEffect !== null);

    nextEffect = null;

    // Tell Scheduler to yield at the end of the frame, so the browser has an
    // opportunity to paint.
    requestPaint();
  } else {
    // No effects.
    root.current = finishedWork;
  }

  onCommitRoot(finishedWork.stateNode, expirationTime);

  // If layout work was scheduled, flush it now.
  flushSyncCallbackQueue();
  return null;
}
```

### commitBeforeMutationLifeCycles
> 文件位置：react-reconciler/src/ReactFiberCommitWork.js


before mutation 阶段执行逻辑。
函数调用栈：commitRootImpl -->  commitBeforeMutationEffects --> commitBeforeMutationLifeCycles

主要代码如下：
```javascript
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    if ((nextEffect.effectTag & Snapshot) !== NoEffect) {
      const current = nextEffect.alternate;
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }
    nextEffect = nextEffect.nextEffect;
  }
}

function commitBeforeMutationLifeCycles(
  current: Fiber | null,
  finishedWork: Fiber,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      commitHookEffectList(UnmountSnapshot, NoHookEffect, finishedWork);
      return;
    }
    /*
     class组件执行 instance.getSnapshotBeforeUpdate
    */
    case ClassComponent: {
      if (finishedWork.effectTag & Snapshot) {
        if (current !== null) {
          const prevProps = current.memoizedProps;
          const prevState = current.memoizedState;
          const instance = finishedWork.stateNode;
          // We could update instance props and state here,
          // but instead we rely on them being set during last render.
          // TODO: revisit this when we implement resuming.
 
          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState,
          );
        
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
        }
      }
      return;
    }
    case HostRoot:
    case HostComponent:
    case HostText:
    case HostPortal:
    case IncompleteClassComponent:
      // Nothing to do for these component types
      return;
    default: {
      invariant(
        false,
        'This unit of work tag should not have side-effects. This error is ' +
          'likely caused by a bug in React. Please file an issue.',
      );
    }
  }
}
```

### commitBeforeMutationLifeCycles
> 文件位置：react-reconciler/src/ReactFiberWorkLoop.js


mutation phase 阶段执行逻辑。
函数调用栈：commitRootImpl -->  commitMutationEffects --> commitPlacement 或 commitWork 或 commitDeletion等

主要代码如下：
```javascript
function commitMutationEffects() {
  // TODO: Should probably move the bulk of this function to commitWork.
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag;

    // The following switch statement is only concerned about placement,
    // updates, and deletions. To avoid needing to add a case for every possible
    // bitmap value, we remove the secondary effects from the effect tag and
    // switch on that value.
    let primaryEffectTag = effectTag & (Placement | Update | Deletion);
    switch (primaryEffectTag) {
      case Placement: {
        commitPlacement(nextEffect);
        // Clear the "placement" from effect tag so that we know that this is
        // inserted, before any life-cycles like componentDidMount gets called.
        // TODO: findDOMNode doesn't rely on this any more but isMounted does
        // and isMounted is deprecated anyway so we should be able to kill this.
        nextEffect.effectTag &= ~Placement;
        break;
      }
      case PlacementAndUpdate: {
        // Placement
        commitPlacement(nextEffect);
        // Clear the "placement" from effect tag so that we know that this is
        // inserted, before any life-cycles like componentDidMount gets called.
        nextEffect.effectTag &= ~Placement;

        // Update
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Deletion: {
        commitDeletion(nextEffect);
        break;
      }
    }
  }
}
```

### commitLayoutEffects
> 文件位置：react-reconciler/src/ReactFiberCommitWork.js


layout phase 阶段执行逻辑。
函数调用栈：commitRootImpl -->  commitLayoutEffects --> commitLifeCycles 

主要代码如下：
```javascript
function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedExpirationTime: ExpirationTime,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      commitHookEffectList(UnmountLayout, MountLayout, finishedWork);
      break;
    }
    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (finishedWork.effectTag & Update) {
        if (current === null) {
          instance.componentDidMount();
        } else {
          const prevProps =
            finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps);
          const prevState = current.memoizedState;
          instance.componentDidUpdate(
            prevProps,
            prevState,
            instance.__reactInternalSnapshotBeforeUpdate,
          );
        }
      }
      const updateQueue = finishedWork.updateQueue;
      if (updateQueue !== null) {
        // We could update instance props and state here,
        // but instead we rely on them being set during last render.
        // TODO: revisit this when we implement resuming.
        commitUpdateQueue(
          finishedWork,
          updateQueue,
          instance,
          committedExpirationTime,
        );
      }
      return;
    }
    case HostRoot: {
      const updateQueue = finishedWork.updateQueue;
      if (updateQueue !== null) {
        let instance = null;
        if (finishedWork.child !== null) {
          switch (finishedWork.child.tag) {
            case HostComponent:
              instance = getPublicInstance(finishedWork.child.stateNode);
              break;
            case ClassComponent:
              instance = finishedWork.child.stateNode;
              break;
          }
        }
        commitUpdateQueue(
          finishedWork,
          updateQueue,
          instance,
          committedExpirationTime,
        );
      }
      return;
    }
    case HostComponent: {
      const instance: Instance = finishedWork.stateNode;

      // Renderers may schedule work to be done after host components are mounted
      // (eg DOM renderer may schedule auto-focus for inputs and form controls).
      // These effects should only be committed when components are first mounted,
      // aka when there is no current/alternate.
      if (current === null && finishedWork.effectTag & Update) {
        const type = finishedWork.type;
        const props = finishedWork.memoizedProps;
        commitMount(instance, type, props, finishedWork);
      }

      return;
    }
    case HostText: {
      // We have no life-cycles associated with text.
      return;
    }
    case HostPortal: {
      // We have no life-cycles associated with portals.
      return;
    }
    case Profiler: {
      if (enableProfilerTimer) {
        const onRender = finishedWork.memoizedProps.onRender;

        if (enableSchedulerTracing) {
          onRender(
            finishedWork.memoizedProps.id,
            current === null ? 'mount' : 'update',
            finishedWork.actualDuration,
            finishedWork.treeBaseDuration,
            finishedWork.actualStartTime,
            getCommitTime(),
            finishedRoot.memoizedInteractions,
          );
        } else {
          onRender(
            finishedWork.memoizedProps.id,
            current === null ? 'mount' : 'update',
            finishedWork.actualDuration,
            finishedWork.treeBaseDuration,
            finishedWork.actualStartTime,
            getCommitTime(),
          );
        }
      }
      return;
    }
    case SuspenseComponent:
    case SuspenseListComponent:
    case IncompleteClassComponent:
      return;
    case EventComponent: {
      if (enableFlareAPI) {
        mountEventComponent(finishedWork.stateNode);
      }
      return;
    }
    default: {
      invariant(
        false,
        'This unit of work tag should not have side-effects. This error is ' +
          'likely caused by a bug in React. Please file an issue.',
      );
    }
  }
}
```

## 扩展
其他一些关于fiber的讨论

### 关于requestIdleCallback
之前有文章总结react的调度原理时提到，客户端线程执行任务时会以帧的形式划分，在两个执行帧之间，主线程通常会有一小段空闲时间，requestIdleCallback可以在这个空闲期调用空闲期回调，执行一些优先级较低的work。

在早期的React版本上是这么做的，但requestIdleCallback实际上有一些限制，而且执行频次不足，以致于无法实现流畅的UI渲染，所以React团队放弃了requestIdleCallback用法，不得不实现自己的版本，详情可以查看react源码 packages/scheduler部分

### Stack Reconciler 和 fiber reconciliation
Stack Reconciler是React v15及之前版本使用的协调算法。

Stack Reconciler的实现使用了同步递归模型，该模型依赖于内置堆栈来遍历树。它具有一定的限制性，最大的地方在于，无法将work分解成增量单位。我们不能在某个特定组件的暂停work，并在之后继续执行。

注意，在stack reconciler下，DOM的更新是同步的，也就是说，在virtual DOM的比对过程中，发现一个instance有更新，会立即执行DOM操作。

那么fiber reconciliation如何实现无需递归遍历树的算法呢? 它使用单链表树遍历算法，这使得暂停遍历和停止堆栈递归成为可能。

React 团队的Andrew提到：
> 如果只依赖内置调用堆栈，那么它将一直工作，直到堆栈为空，如果我们可以随意中断调用堆栈并手动操作堆栈帧，这不是很好吗?这就是React Fiber的目标。fiber是内置堆栈的重新实现，专门用于React组件，可以将一个fiber看作是一个虚拟堆栈帧。


因此，为了解决这个问题，React必须重新实现从依赖于内置堆栈的同步递归模型到包含链表和指针的异步模型的树遍历算法。重新实现堆栈的优点是，可以将堆栈帧保留在内存中，在任何时候执行它们，这对于实现调度目标至关重要。




## 参考
《[facebook](https://github.com/facebook)/[react](https://github.com/facebook/react)[](https://github.com/facebook/react)》[https://github.com/facebook/react](https://github.com/facebook/react)
《In-depth explanation of state and props update in React》[https://medium.com/react-in-depth/in-depth-explanation-of-state-and-props-update-in-react-51ab94563311](https://medium.com/react-in-depth/in-depth-explanation-of-state-and-props-update-in-react-51ab94563311)
《in-depth overview of the new reconciliation algorithm in React》[https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e)
《The how and why on React’s usage of linked list in Fiber to walk the component’s tree》[https://medium.com/react-in-depth/in-depth-explanation-of-state-and-props-update-in-react-51ab94563311](https://medium.com/react-in-depth/in-depth-explanation-of-state-and-props-update-in-react-51ab94563311)
《effect-list fiber链表构建过程》[https://www.bilibili.com/video/av48384879/](https://www.bilibili.com/video/av48384879/)
《[react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture)》[https://github.com/acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture)
