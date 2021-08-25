> - 原文链接：https://indepth.dev/posts/1009/in-depth-explanation-of-state-and-props-update-in-react
> - 原文标题：In-depth explanation of state and props update in React
> - 原文作者：[Max Koretskyi](https://twitter.com/maxkoretskyi)

在我的上篇文章 [Inside Fiber: 深入了解React新协调算法](https://indepth.dev/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)中介绍了理解更新过程细节的所需的基础知识，我将在本文中描述这个更新过程。

我已经概述了将在本文中使用的主要数据结构和概念，特别是Fiber节点，`current`和`work-in-progress`树，副作用（side-effects）以及effects链表（effects list）。我也提供了主要算法的高级概述和`render`阶段与`commit`阶段的差异。如果你还没有阅读过它，我推荐你从那儿开始。

我还向你介绍了带有一个按钮的示例程序，这个按钮的功能就是简单的增加数字。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bec3f434e7ca4322a0678d7a049f7f75~tplv-k3u1fbpfcp-zoom-1.image)

你可以在这查看[在线代码](https://stackblitz.com/edit/react-jwqn64)。它的实现很简单，就是一个render函数中返回`button`和`span`元素的类组件。当你点击按钮的时候，在点击事件的处理函数中更新组件的`state`。结果就是`span`元素的文本会更新。
```js
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
我为这个组件添加了`componentDidUpdate`生命周期方法。这是为了演示React如何添加[effects](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e)并在`commit`阶段调用这个方法。在本文中，我想向你展示React是如何处理状态更新和创建effects list的。我们可以看到`render`阶段和`commit`阶段的高级函数中发生了什么。

尤其是在React的[completeWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberCompleteWork.js#L532)函数中：
- 更新`ClickCounter`的`state`中的`count`属性
- 调用`render`方法获取子元素列表并比较
- 更新`span`元素的`props`
以及，在React的[commitRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523) 函数中：
- 更新`span`元素的文本内容属性
- 调用`componentDidUpdate`生命周期方法

但是在那之前，我们先快速看看当我们在点击处理函数中调用`setState`时工作是如何调度的。

**请注意，你无需了解这些来使用React。本文是关于React内部是如何运作的。**
## 调度更新

当我们点击按钮时，`click`事件被触发，React执行传递给按钮`props`的回调。在我们的程序中，它只是简单的增加计数器和更新`state`。

```js
class ClickCounter extends React.Component {
    ...
    handleClick() {
        this.setState((state) => {
            return {count: state.count + 1};
        });
    }
}   
```
每个组件都有相应的`updater`，它作为组件和React核心之间的桥梁。这允许`setState`在ReactDOM，React Native，服务端渲染和测试程序中是不同的实现。（译注：从[源码](https://github.com/facebook/react/blob/master/packages/react/src/ReactBaseClasses.js#L65)可以看出，`setState`内部是调用`updater.enqueueSetState`，这样在不同平台，我们都可以调用`setState`来更新页面）

本文中，我们关注ReactDOM中实现的`updater`对象，它使用Fiber协调器。对于`ClickCounter`组件，它是[classComponentUpdater](https://github.com/facebook/react/blob/6938dcaacbffb901df27782b7821836961a5b68d/packages/react-reconciler/src/ReactFiberClassComponent.js#L186)。它负责获取Fiber实例，为更新入列，以及调度work。

当更新排队时，它们基本上只是添加到Fiber节点的更新队列中进行处理。在我们的例子中，`ClickCounter`组件对应的[Fiber节点](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e)将有下面的结构：

```js
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
如你所见，`updateQueue.firstUpdate.next.payload`中的函数就我我们在`ClickCounter`组件中传递给`setState`的回调。它代表在`render`阶段中需要处理的第一个更新。

## 处理ClickCounter Fiber节点的更新

我上篇文章中的[work循环部分中](https://indepth.dev/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)解释了全局变量`nextUnitOfWork`的角色。尤其是，这个变量保存`workInProgress`树中有work待做的Fiber节点的引用。当React遍历树的Fiber时，它使用这个变量知道是否存在其他有未完成work的Fiber节点。

我们假定`setState`方法已经被调用。 React将setState中的回调添加到`ClickCounter`fiber节点的`updateQueue`中，然后调度work。React进入`render`阶段。它使用[renderRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1132)函数从最顶层`HostRoot`Fiber节点开始遍历。然而，它会跳过已经处理过得Fiber节点直到遇到有未完成work的节点。基于这点，只有一个节点有work待做。它就是`ClickCounter`Fiber节点。

所有的work都是基于保存在Fiber节点的`alternate`字段的克隆副本执行的。如果alternate节点还未创建，React在处理更新前调用[createWorkInProgress](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L326)函数创建副本。我们假设`nextUnitOfWork`变量保存代替`ClickCounter`Fiber节点的引用。

## beginWork
首先, 我们的Fiber进入[beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489)函数。

> 因为这个函数对树中每个节点执行，所以如果你想调试render阶段，它是放置断点的好地方。 我经常这样做，还有检查Fiber节点的type来确定我需要的节点。

`beginWork`函数大体上是个大的`switch`语句，通过`tag`确定Fiber节点需要完成的work的类型，然后执行相应的函数来执行work。在这个例子中，`CountClicks`是类组件，所以会走这个分支：

```js
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
我们进入[updateClassComponent](https://github.com/facebook/react/blob/1034e26fe5e42ba07492a736da7bdf5bf2108bc6/packages/react-reconciler/src/ReactFiberBeginWork.js#L428)函数。取决于它是首次渲染、恢复work还是React更新，React会创建实例并挂载组件或只是更新它：
```js
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

## 处理ClickCounter Fiber更新
我们已经有了`ClickCounter`组件实例，所以我们进入[updateClassInstance](https://github.com/facebook/react/blob/6938dcaacbffb901df27782b7821836961a5b68d/packages/react-reconciler/src/ReactFiberClassComponent.js#L976)。这是React为类组件执行大部分work的地方。以下是在这个函数中按顺序执行的最重要的操作：
- 调用`UNSAFE_componentWillReceiveProps()`钩子（已废弃）
- 处理`updateQueue`中的更新以及生成新state
- 使用新state调用`getDerivedStateFromProps`并得到结果
- 调用`shouldComponentUpdate`确定组件是否需要更新；如果返回结果为`false`，跳过整个渲染过程，包括在该组件和它的子组件上调用`render`；否则继续更新
- 调用`UNSAFE_componentWillUpdate`（已废弃）
- 添加一个effect来触发`componentDidUpdate`生命周期钩子
> 尽管调用`componentDidUpdate`的effect是在`render`阶段添加的，这个方法将在接下来的`commit`阶段执行。
- 更新组件实例的`state`和`props`
组件实例的`state`和`props`应该在`render`方法调用前更新，因为`render`方法的输出通常依赖于`state`和`props`。如果我们不这样做，它每次都会返回一样的输出。

下面是该函数的简化版本：

```js
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
上面代码片段中我删除了一些辅助代码。对于实例，调用生命周期方法或添加effects来触发它们前，React使用`typeof`操作符检查组件是否实现了这些方法。比如，这是React添加effect前如何检查`componentDidUpdate`方法：
```js
if (typeof instance.componentDidUpdate === 'function') {
    workInProgress.effectTag |= Update;
}
```
好的，我们现在知道了`render`阶段中为`ClickCounter`Fiber节点执行了什么操作。现在让我们看看这些操作如何改变Fiber节点的值。当React开始work，`ClickCounter`组件的Fiber节点类似这样：
```js
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
```
work完成后，我们得到一个长这样的Fiber节点：
```js
{
    effectTag: 4,
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
**花点时间观察属性值的差异**

更新被应用后，`memoizedState`和`updateQueue`中`baseState`的属性`count`的值变为`1`。React也更新了`ClickCounter`组件实例的state。

至此，队列中不再有更新，所以`firstUpdate`为`null`。更重要的是，我们改变了`effectTag`属性。它不再是`0`，它的是为`4`。 二进制为`100`，意味着第三位被设置了，代表`Update`[副作用标记](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js)：

```js
export const Update = 0b00000000100;
```
可以得出结论，当执行`ClickCounter`Fiber节点的work时，React低啊用变化前生命周期方法，更新state，定义有关的副作用。

## 协调ClickCounter Fiber的子组件

在那之后，React进入[finishClassComponent](https://github.com/facebook/react/blob/340bfd9393e8173adca5380e6587e1ea1a23cefa/packages/react-reconciler/src/ReactFiberBeginWork.js#L355)。这是调用组件实例render方法和在子组件上使用diff算法的地方。[文档中](https://reactjs.org/docs/reconciliation.html#the-diffing-algorithm)对此有高级概述。以下是相关部分：
> 当比较两个相同类型的React DOM元素时，React查看两者的属性（attributes），保留DOM节点，仅更新变化了的属性。

然而，如果我们深入挖掘，会知道它实际是对比Fiber节点和React元素。但是我现在不会详细介绍因为过程相当复杂。我会单独些篇文章，特别关注子协调过程。
> 如果你想自己学习细节，请查看[reconcileChildrenArray](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactChildFiber.js#L732)函数，因为在我们的程序中`render`方法返回一个React元素数组。

至此，有两个很重要的事需要理解。**第一**，当React进行子协调时，它会为从`render`函数返回的子React元素创建或更新Fiber节点。`finishClassComponent`函数当前Fiber节点的第一个子节点的引用。它被赋值给`nextUnitOfWork`并在稍后的work循环中处理。**第二**，React更新子节点的props作为父节点执行的一部分work。为此，它使用render函数返回的React元素的数据。

举例来说，这是React协调`ClickCounter`fiber子节点之前`span`元素对应的Fiber节点看起来的样式

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    key: "2",
    memoizedProps: {children: 0},
    pendingProps: {children: 0},
    ...
}
```
可以看到，`memoizedProps`和`pendingProps`的`children`属性都是`0`。这是`render`函数返回的`span`元素对应的React元素的结构。

```js
{
    $$typeof: Symbol(react.element)
    key: "2"
    props: {children: 1}
    ref: null
    type: "span"
}
```
可以看出，Finer节点和返回的React元素的**props是有差异的**。[createWorkInProgress](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L326)内部用这创建替代的Fiber节点，**React把React元素中更新的属性复制到Fiber节点**。

因此，在React完成`ClickCounter`组件子协调后，`span`的Fiber节点的`pendingProps`更新了。它们将匹配`span`React元素中的值。

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    key: "2",
    memoizedProps: {children: 0},
    pendingProps: {children: 1},
    ...
}
```

稍后，React会为`span`Fiber节点执行work，它将把它们复制到`memoizedProps`以及添加effects来更新DOM。

好的，这就是render阶段React为`ClickCounter`fiber节点所执行的所有work。因为button是`ClickCounter`组件的第一个子节点，它会被赋值给`nextUnitOfWork`变量。button上无事可做，所有React会移动到它的兄弟节点`span`Fiber节点上。根据[这里描述的](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e)算法，这发生在`completeUnitOfWork`函数内。
## 处理Span fiber的更新
`nextUnitOfWork`变量现在指向`span`fiber的alternate，React基于它开始工作。和`ClickCounter`执行的步骤类似，开始于[beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489)函数。

因为`span`节点是`HostComponent`类型，这次在switch语句中React会进入这条分支：

```js
function beginWork(current$$1, workInProgress, ...) {
    ...
    switch (workInProgress.tag) {
        case FunctionalComponent: {...}
        case ClassComponent: {...}
        case HostComponent:
          return updateHostComponent(current, workInProgress, ...);
        case ...
}
```
结束于[updateHostComponent](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L686)函数。（在这个函数内）你可以看到一系列和类组件调用的`updateClassComponent`函数类似的函数。对于函数组件是`updateFunctionComponent`。你可以在[ReactFiberBeginWork.js](https://github.com/facebook/react/blob/1034e26fe5e42ba07492a736da7bdf5bf2108bc6/packages/react-reconciler/src/ReactFiberBeginWork.js)文件中找到这些函数。
## 协调Span fiber子节点
在我们的例子中，`span`节点在`updateHostComponent`里没什么重要事的发生。
## 完成Span Fiber节点的work
一旦`beginWork`完成，节点就进入`completeWork`函数。但是在那之前，React需要更新`span` Fiber节点的`memoizedProps`属性。你应该还记得协调`ClickCounter`组件子节点时更新了`span`Fiber节点的`pendingProps`。

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    key: "2",
    memoizedProps: {children: 0},
    pendingProps: {children: 1},
    ...
}
```
所以一旦`span`fiber的`beginWork`完成，React会将`pendingProps`更新到`memoizedProps`。

```js
function performUnitOfWork(workInProgress) {
    ...
    next = beginWork(current$$1, workInProgress, nextRenderExpirationTime);
    workInProgress.memoizedProps = workInProgress.pendingProps;
    ...
}
```
然后调用的`completeWork`和我们看过的`beginWork`相似，基本上是一个大的switch语句。

```js
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
由于`span`Fiber节点是`HostComponent`，它会执行[updateHostComponent](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L686)函数。在这个函数中React大体上做了这些事：

- 准备DOM更新
- 把它们加到`span`fiber的`updateQueue`
- 添加effect用于更新DOM

在这些操作执行前，`span`Fiber节点看起来像这样：

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    effectTag: 0
    updateQueue: null
    ...
}
```
works完成后它看起来像这样：

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    effectTag: 4,
    updateQueue: ["children", "1"],
    ...
}
```
注意`effectTag`和`updateQueue`字段的差异。它不再是`0`，它的值是`4`。用二进制表示是`100`，意味着设置了第3位，正是`Update`副作用的标志位。这是React在接下来的commit阶段对这个节点唯一要做的任务。`updateQueue`保存着用于更新的载荷。

一旦React处理完`ClickCounter`级它的子节点，`render`阶段结束。现在它可以将完成的替代树赋值给`FiberRoot`的`finishedWork`属性。这是需要被刷新到屏幕上的新树。它可以在`render`阶段之后马上被处理，或这当React被浏览器给予时间时再处理。
## Effects list
在我们的例子中，由于`span`节点`ClickCounter`组件有副作用，React将添加指向`span`Fiber节点的链接到`HostFiber`的`firstEffect`属性。

React在[compliteUnitOfWork](https://github.com/facebook/react/blob/d5e1bf07d086e4fc1998653331adecddcd0f5274/packages/react-reconciler/src/ReactFiberScheduler.js#L999)函数内构建effects list。这是带有更新`span`节点文本和调用`ClickCounter`上hooks副作用的Fiber树看起来的样子：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ccc1deb97dc543abaca5c0a979eaf938~tplv-k3u1fbpfcp-zoom-1.image)

这是由有副作用的节点组成的线性列表：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c39fef6347fd4c3cb64b606d64f51f0e~tplv-k3u1fbpfcp-zoom-1.image )

## Commit阶段
这个阶段开始于[completeRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L2306)函数。它在做其他工作之前，它将`FiberRoot`的`finishedWork`属性设为`null`：

```js
root.finishedWork = null;
```
于之前的`render`阶段不同的是，`commit`阶段总是同步的，这样它可以安全地更新`HostRoot`来表示commit work开始了。

`commit`阶段是React更新DOM和调用突变后生命周期方法`componentDidUpdate`的地方。为此，它遍历在`render`阶段中构建的effects list并应用它们。

有以下在`render`阶段为`span`和`ClickCounter`定义的effects：

```js
{ type: ClickCounter, effectTag: 5 }
{ type: 'span', effectTag: 4 }
```
`ClickCounter`的effect tag的值是`5`或二进制的`101`，定义了对于类组件基本上转换为`componentDidUpdate`生命周期方法的`Update`工作。最低位也被设置了，表示该Fiber节点在`render`阶段的所有工作都已完成。

`span`的effect tag的值是`4`或二进制的`100`，定义了原生组件DOM更新的`update`工作。这个例子中的`span`元素，React需要更新这个元素的`textContent`。
## 应用effects
让我们看看React如何应用这些effects。[commitRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523)函数用于应用这些effects，由3个子函数组成：

```js
function commitRoot(root, finishedWork) {
    commitBeforeMutationLifecycles()
    commitAllHostEffects();
    root.current = finishedWork;
    commitAllLifeCycles();
}
```
每个子函数都实现了一个循环，该循环用于遍历effects list并检查这些effects的类型。当发现effect和函数的目的有关时就应用它。我们的例子中，它会调用`ClickCounter`组件的`componentDidUpdate`生命周期方法，更新`span`元素的文本。

第一个函数 [commitBeforeMutationLifeCycles](https://github.com/facebook/react/blob/fefa1269e2a67fa5ef0992d5cc1d6114b7948b7e/packages/react-reconciler/src/ReactFiberCommitWork.js#L183) 寻找 [Snapshot](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js#L25) effect然后调用`getSnapshotBeforeUpdate`方法。但是，我们在`ClickCounter`组件中没有实现该方法，React在`render`阶段没有添加这个effect。所以在我们的例子中，这个函数不做任何事。
## DOM更新
接下来React执行 [commitAllHostEffects](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L376) 函数。这儿是React将`span`元素的t文本由`0`变为`1`的地方。`ClickCounter` fiber没有要做的，因为类组件的节点没有任何DOM更新。

这个函数的主旨是选择正确类型的effect并应用相应的操作。在我们的例子中我们需要跟新`span`元素的文本，所以我们采用`Update`分支：

```js
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
```
随着`commitWork`执行，最终会进入[updateDOMProperties](https://github.com/facebook/react/blob/8a8d973d3cc5623676a84f87af66ef9259c3937c/packages/react-dom/src/client/ReactDOMComponent.js#L326)函数。它使用在`render`阶段添加到Fiber节点的`updateQueue`载荷更新`span`元素的`textContent`。

```js
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
```
应用DOM更新后，React将`finishedWork`赋值给`HostRoot`。它将替代树是设为当前树：

```js
root.current = finishedWork;
```
## 调用突变后生命周期hooks

剩下的函数是[****commitAllLifecycles****](https://github.com/facebook/react/blob/d5e1bf07d086e4fc1998653331adecddcd0f5274/packages/react-reconciler/src/ReactFiberScheduler.js#L479)。这是 React 调用突变后生命周期方法的地方。在`render`阶段，React为`ClickCounter`组件添加`Update` effect。这是`commitAllLifecycles`寻找的effects之一并调用`componentDidUpdate`方法：

```js
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
```
这个函数也更新[refs](https://reactjs.org/docs/refs-and-the-dom.html)，但是由于我们没有使用这个特性，所以没什么作用。这个方法在[commitLifeCycles](https://github.com/facebook/react/blob/e58ecda9a2381735f2c326ee99a1ffa6486321ab/packages/react-reconciler/src/ReactFiberCommitWork.js#L351)函数中被调用：

```js
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
也可以看出，这是首次渲染时React调用组件`componentDidMount`方法的函数。