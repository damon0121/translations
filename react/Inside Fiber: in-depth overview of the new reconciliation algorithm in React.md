> - 原文链接：https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react 
> - 原文标题：Inside Fiber: in-depth overview of the new reconciliation algorithm in React
> - 原文作者：[Max Koretskyi](https://twitter.com/maxkoretskyi)

`React`是一个用于构建用户界面的`JavaScript`库。它的核心机制是跟踪组件的`state`变化并将更新后的`state`显示到屏幕上。在React中这个过程叫做**协调**(**reconciliation**)。我们调用`setState`方法，React会检查`state`或`props`是否变化，然后重新渲染组件到UI上。

React文档为这个机制提供了[很好的高级概述](https://reactjs.org/docs/reconciliation.html)：React元素的角色，生命周期方法和`render`方法，以及应用到组件`children`的`diffing`算法。由`render`方法返回的元素组成的树通常被认为是”虚拟DOM“。这个术语早期有助于理解`React`，但是它也引起了困惑并且在`React`文档里已经不再使用它了。在这篇文章里我称其为`React`元素树。

除了`React`元素树，还有一颗内部实例树（组件，DOM节点等等）用于保存状态。从版本16开始，React推出了内部树和管理内部树的算法的实现，称为Fiber。通过[React如何以及为什么在Fiber中使用链表 ](https://indepth.dev/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-to-walk-the-components-tree/)。

这是让你了解`React`内部架构系列的第一篇文章。在这篇文章中，我想提供关于这个算法的重要概念和数据结构的概述。一旦我们拥有足够的背景知识，我们就会探索该算法用于遍历和处理`fiber`树的主要函数。在这个系列接下来的文章中会展示React是如何使用这个算法进行首次渲染，处理`state`和`props`的更新。在那之前，我们先了解调度器、协调过程和构建`effects`列表的机制的细节。

我会教你一些相当高级的知识？我鼓励你阅读它来理解`Concurrent React`内部运作背后的魔法。如果你想为`React`贡献，这个系列的文章可以作为你很好的指南。我[坚信逆向工程](https://indepth.dev/level-up-your-reverse-engineering-skills/)，所以会有很多版本16.6.0的源码链接。

这确实要花费大量时间和精力，所以不要气馁即使你不能马上理解。花费时间是值得的。**注意，你无需知道这些也能使用`React`，这篇文章是关于`React`内部是如何运作的。**
# 背景设定
这是一个我准备贯穿整个系列的简单程序。在屏幕上我们有个简单增加数字的按钮：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb8bd8632f804939b0563ad8d1e876de~tplv-k3u1fbpfcp-zoom-1.image)

这是实现：

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


    render() {
        return [
            <button key="1" onClick={this.handleClick}>Update counter</button>,
            <span key="2">{this.state.count}</span>
        ]
    }
}
```
你可以在[这](https://stackblitz.com/edit/react-t4rdmh)查看。如你所见，它是一个`render`方法中返回两个子元素`button`和`span`的简单组件。一旦你点击按钮，组件的`state`在处理函数中被更新。这样就会导致`span`元素的文本更新。

在`React`的**协调**过程中有很多活动，比如调用[ 生命周期方法 ](https://reactjs.org/docs/react-component.html#updating)、更新[refs](https://reactjs.org/docs/refs-and-the-dom.html)。**在`Fiber`架构中这些活动都称为”work“**。work的类型通常依赖于`React`元素的类型。举个例子，对于类组件，`React`需要创建一个实例，而函数组件则不必这样。正如你所知道的，`React`中有很多种类的元素，如类组件和函数组件，原生组件（DOM节点），portals等等。`React`元素的类型是由[createElement](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/react/src/ReactElement.js#L171)函数的第一个参数确定的。这个函数通常用于`render`方法中用来创建一个元素。

在研究这些活动和`fiber`主要算法前，我们先来熟悉`React`内部使用的数据结构。
# 从`React`元素到`Fiber`节点
`React`的每个组件都有UI表示，我们可以称从`render`方法返回的为视图或模板。这是我们`ClickCounter`组件的模板：

```js
<button key="1" onClick={this.onClick}>Update counter</button>
<span key="2">{this.state.count}</span>
```
## React元素
一个模板经过JSX编译器编译后，就会得到一堆`React`元素。这才是真正从`render`中返回的东西，而不是`HTML`。如果不适用JSX，我们`ClickCounter`组件的`render`方法应该写成这样：

```js
class ClickCounter {
    ...
    render() {
        return [
            React.createElement(
                'button',
                {
                    key: '1',
                    onClick: this.onClick
                },
                'Update counter'
            ),
            React.createElement(
                'span',
                {
                    key: '2'
                },
                this.state.count
            )
        ]
    }
}
```
`render`方法中调用`React.createElement`会创建像这样的数据结构：

```js
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
可以看到，`React`为这些对象添加了[$$typeof](https://overreacted.io/why-do-react-elements-have-typeof-property/)属性来表示它们是`React`元素。还有些属性`type`、`key`和`props`来描述元素。这些值是通过`React.createElement`函数传递进来的。注意`React`如何让文本内容作为`span`和`button`的`children`。以及点击事件如何作为`button`元素的`props`的一部分。`React`元素上还有其他一些超出本文讨论范围的字段比如`ref`。

`ClickCounter`的`React`元素没有任何`props`或`ref`
```js
{
    $$typeof: Symbol(react.element),
    key: null,
    props: {},
    ref: null,
    type: ClickCounter
}
```
## Fiber节点
在**协调**过程中，每个从`render`返回的`React`元素会被合并一颗`fiber`树。每个`React`元素都有相应的`fiber`节点。与`React`元素不同的是，`fiber`节点不会再每次渲染是从新创建。它们是可变的数据结构，保存了组件`state`和DOM。

我们之前讨论过`React`根据元素类型执行不同活动。在我们的实例程序中，对于类组件`ClickCounter`会调用生命周期方法和`render`方法，而对于`span`这样的原生组件（DOM节点）会执行DOM变化。所以每个React元素会被转换成[相应类型](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/shared/ReactWorkTags.js)的Fiber节点，这个节点描述了需要完成的work。

**你可以将`Fiber`理解为一种表示待做的一些work的数据结构，或者换句话说，一个work单元。Fiber架构也提供了一种方便的方式来追踪、调度、暂停和中止这些work。**

当一个`React`元素第一次转换成`fiber`节点时，`React`在[createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414)函数中使用元素中的数据来创建一个`fiber`。在更新中`React`会复用fiber节点，根据相应的`React`元素仅更新必要的属性。React也可能根据`key`prop来移动节点，或者如果相应的的`React`不再从`render`方法中返回，那么就删除它。
> 查看[ChildReconciler](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactChildFiber.js#L239)函数来了解活动列表以及`React`对于已经存在的`fiber`节点执行的函数。

因为`React`为每个`React`元素创建了fiber节点并且我们有一颗由这些元素组成的树，所以我们将有一个由`fiber`节点组成的树。在我们例子中看起来像这样：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26a024409d894c8580f6db69101d3eb4~tplv-k3u1fbpfcp-zoom-1.image)

所有`fiber`节点都是通过`fiber`节点上的这几个属性形成链表：`child`，`subling`和`return`。要了解为什么这样做的更多细节，请阅读我的文章[React如何以及为什么在Fiber中使用链表](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)，如果你还没读过。

## Current和work in process树
首次渲染后，`React`中存在一颗保存了应用程序状态，用于渲染UI的`fiber`树。这颗树通常称为**current**。当React开始进行更新时，它创建一颗所谓的`workInProgress`树，这棵树保存着将来要刷新到屏幕上的状态。

所有的work都是在`workInProgress`树的fibers上执行的。当`React`遍历`current`树，对于每个现存的fiber节点，React会创建一个代替（alternate）节点，这些代替节点组成`workInProgress`树。代替节点是由`render`方法返回的`React`元素的数据创建的。一旦更新都被处理了、所有相关联的work完成了，`React`就会有一颗准备刷新到屏幕上的代替树。一旦`workInProgress`树渲染到屏幕上，它就变成`current`树。

React的核心原则之一就是连贯性。React总是一次性更新DOM，它不会显示部分结果。`workInProgress`树就像一份草稿，用户是看不见它的，所以React可以先处理所有组件，然后在将它们的变化更新到屏幕上。

在源码中你会看到很多使用`current`和`workInProgress`树节点的函数，这是其中一个函数的签名：

```js
function updateHostComponent(current, workInProgress, renderExpirationTime) {...}
```
每个fiber节点在**alternate**字段上保存了另一颗树上相应节点的引用。`current`树上节点指向`workInProgress`树上相应的节点，反之亦然。
## Side-effects（副作用）
我们可以把React中的组件看成一个使用`state`和`props`来得到UI页面的函数。其他的每个活动比如DOM变化或调用生命周期方法都应该被认为是副作用或作用。Effects[在文档中](https://reactjs.org/docs/hooks-overview.html#%EF%B8%8F-effect-hook)也有提及。
> 你之前在React组件中可能执行过数据获取，订阅，或手动**更改DOM**。我们把这些操作称为副作用（或简称作用），因为它们可能影响其他组件，而且在渲染过程中无法完成。

你可以看到大部分`state`和`props`更新如何产生副作用。由于标记effects是一种work，除了更新外，fiber节点是一种方便跟踪effects的机制。每个fiber节点都可以关联它的effects。它们保存在`effectTag`字段上。

因此，Fiber中的effects基本定义了更新被处理后实例需要完成的[work](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js)。对于原生组件（DOM元素），work包含添加、更新或移除元素。对于类组件，React可能需要更新`refs`，调用`componentDidMount`和`componentDidUpdate`生命周期方法。其他类型的fibers有相应的其他effects。

## Effects list
React处理更新非常快，为了实现高性能它使用了一些有趣的技术。**它们中的一个就是创建一个由包含effects的fiber节点组成的线性链表来实现快速迭代。** 迭代线链列表比迭代一颗树快的多，而且无需在没有副作用的节点上浪费时间。

这个链表的目标是标记含有DOM更新或其他effects的节点并把它们关联起来。这个链表是`finishedWork`树的子集，节点之间使用`nextEffect`属性进行连接，而不是 `current`和`workInProgress`树中使用的 `child`属性。

[Dan Abramov](https://medium.com/u/a3a8af6addc1?source=post_page---------------------------)为effects list描述了一种比喻。他喜欢将它想象成挂在圣诞树上的”圣诞灯“，”圣诞灯“将所有有副作用的节点绑到一起。形象点说，把下面fiber树种高亮的节点想象成有些work要做的节点。比如，我们的更新导致`c2`插入DOM中，`d2`和`c1`改变了属性，`b2`触发了一个生命周期方法。effects list会把它们连接起来，如此，React在后面就可以跳过其他节点：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/416598ccecd54e41af27a5206999fe46~tplv-k3u1fbpfcp-zoom-1.image)

你可以看到有副作用的节点是如何连接到一起的。遍历节点时，React使用`firstEffect`指针找出list从哪开始。所以上面的图可以看成这样的线性链表：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d793a5c04c2048b293baf7ee2ca820ee~tplv-k3u1fbpfcp-zoom-1.image)

## Root of the fiber tree
每个React程序都有一个或多个DOM元素作为容器。在我们的例子中它是ID为`container`的`div`元素。

```js
const domContainer = document.querySelector('#container');
ReactDOM.render(React.createElement(ClickCounter), domContainer);
```
React为这些容器创建了一个[fiber root](https://github.com/facebook/react/blob/0dc0ddc1ef5f90fe48b58f1a1ba753757961fc74/packages/react-reconciler/src/ReactFiberRoot.js#L31)对象。你可以使用DOM元素的引用来获取它。

```js
const fiberRoot = query('#container')._reactRootContainer._internalRoot
```
这个fiber root就是React保存一颗fiber树引用的地方。它保存在fiber root的`current`属性中。

```js
const hostRootFiberNode = fiberRoot.current
```
fiber树开始于[一个特殊类型](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/shared/ReactWorkTags.js#L34)的fiber节点，它就是`HostRoot`。它在内部创建，作为你最顶层组件的父级。`HostRoot`fiber节点上有个指回`FiberRoot`的`stateNode`属性：

```js
fiberRoot.current.stateNode === fiberRoot; // true
```
你可以通过访问最顶层`HostRoot`fiber节点到达fiber root，接着探索fiber树。
或者你可以从组件实例中获取一个fibe节点，就像这样：
```js
compInstance._reactInternalFiber
```
## Fiber节点结构
现在让我们来看看为`ClickCounter`组件创建的fiber节点的结构：

```js
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    alternate: null,
    key: null,
    updateQueue: null,
    memoizedState: {count: 0},
    pendingProps: {},
    memoizedProps: {},
    tag: 1,
    effectTag: 0,
    nextEffect: null
}
```
`span`DOM元素的fiber节点：

```js
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
fiber节点上有很多字段。在之前的部分中我已经描述过字段`alternate`，`effectTag`，`nextEffect`的作用。现在来看看为什么需要其他字段。
## stateNode
保存类组件实例，DOM节点，或其他与fiber节点关联的React元素类型。总的来说，我们可以说这个属性用于保存与fiber节点关联的本地状态。
## type
定义与这个fiber关联的函数或类。对于类组件，它指向构造函数，对于DOM元素，它代表HTML标签。我经常使用这个字段来理解与一个fiber节点关联的元素是什么。
## tag
定义[the type of the fiber](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/shared/ReactWorkTags.js)。它在协调算法中用于确定什么work要做。如前所述，React元素类型不同，work有所不同。[createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414)函数将React元素映射成相应的fiber节点类型。在我们的程序中，`ClickCounter`组件的`tag`属性是1，表示它是一个`ClassComponent`，`span`元素的是5表示它是一个`HostComponent`。
## updateQueue
一条state更新，回调和DOM更新的队列。
## memoizedState
fiber中用于创建输出的state。当处理更新时，它表示当前渲染到屏幕上的state。
## memoizedProps
在之前渲染中fiber用于创建输出的props。
## pendingProps
React元素中从新数据中更新的props，需要传递给子组件或DOM元素。
## key
一组子元素中的唯一标识符，帮助React从列表中找出哪些项目已变化、添加或者删除。它与React文档[此处](https://reactjs.org/docs/lists-and-keys.html#keys)描述的”列表和keys“功能有关。

你可以在[这](https://github.com/facebook/react/blob/6e4f7c788603dac7fccd227a4852c110b072fe16/packages/react-reconciler/src/ReactFiber.js#L78)看到fiber节点完整的结构。我在上面的说明中删除了一堆字段。特别是我跳过了[我在上篇文章中描述过了](https://indepth.dev/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-to-walk-the-components-tree/)的组成树结构的`child`，`sibling`和`return` 指针。还有一类字段像`expirationTime`，`childExpirationTime`和`mode`，它们是给调度器用的。
# 通用算法
React在两个主要阶段中执行work：**render**和**commit**。

在`render`阶段中，React将更新应用到通过`setState`或`React.render`调度的组件，并且找出什么需要被更新到UI。如果是首次渲染，React为每个从`render`方法中返回的元素创建新的fiber节点。在接下来的更新中，现存的React元素的fiber会被复用和更新。**这个阶段的结果是由标记了副作用的fiber节点组成的树。** effects描述了在接下来的`commit`阶段需要完成的`work`。在这个阶段中，React拥有一颗标记了effects的fiber树，并将它们应用到实例上。它遍历effects链表执行DOM更新和其他用户可见的变化。

**`render`阶段中的work是可以异步执行的，理解这一点很重要。** React在可用时间内处理一个或多个fiber节点，然后停止运行并暂存完成的work，让步于其他事件。然后从它停止的地方继续。但有时，它可能需要放弃已完成的work，再次从顶部开始。正是因为这个阶段执行的work不会导致任何用户可见的变化，比如DOM更新，使得这些暂停成为可能。**相反，后面的`commit`阶段总是同步的。** 这是因为这个阶段执行的work会用户可见的变化，例如DOM更新。这就是为什么React需要一次性完成它们。

调用生命周期方式是React执行的一类work。一些方法在`render`阶段调用，其他的在`commit`阶段调用。下列生命周期函数在`render`阶段中调用：
- [UNSAFE_]componentWillMount (已废弃)
- [UNSAFE_]componentWillReceiveProps (已废弃)
- getDerivedStateFromProps
- shouldComponentUpdate
- [UNSAFE_]componentWillUpdate (已废弃)
- render
如你所见，一些在`render`阶段中执行的遗留的生命周期函数从版本16.3开始被标记为`UNSAFE`。现在再文档中它们被称为遗留的生命周期函数。它们将在16.x发行版中废弃，对应的没有`UNSAFE`前缀的将在17.0中移除。你可以在[这](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html)读到更多关于这些变化和建议的迁移路线。

你对这样做的原因感到好奇吗？

好的，我们刚刚学习了render阶段不会产生像DOM更新这样的副作用，React可以异步处理组件更新（甚至可以在多个线程中运行）。然而，这些被标记为`UNSAFE`被误解和误用。开发人员往往在这些生命周期方法中放入带有副作用的代码，这在新的异步渲染方式中可能引起问题。尽管只有没有`UNSAFE`前缀的会被移除，它们在即将到来的Concurrent模式（你可以选择退出）中仍然可能引起问题。

下列生命周期函数在`commit`阶段中执行：
- getSnapshotBeforeUpdate
- componentDidMount
- componentDidUpdate
- componentWillUnmount
因为执行在同步的`commit`阶段，所以它们可以包含副作用和访问DOM。

好的，现在我们了解了用于遍历树和执行work的算法的背景知识。让我们更深入些。
## Render阶段
协调算法总是从[renderRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1132)函数使用的最顶层`HostRoot`fiber节点开始。然而，React会跳过已经处理了的fiber节点直到它遇到有未完成work的节点。例如，如果你在组件树深层调用`setState`，React将从顶层开始，但是会快速跳过父级直到它到达调用`setState`方法的组件。

### work循环的主要步骤
所有fiber节点在work循环中处理。这是循环的同步部分实现的实现：
```js
function workLoop(isYieldy) {
  if (!isYieldy) {
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {...}
}
```
在上面的代码中，`nextUnitOfWork`保存了来自`workInProgress`树中有work待完成的fiber节点的引用。当React遍历Fiber树时，它使用这个变量来知道是否存在其他有未完成work的fiber节点。当前节点处理完后，这个变量将包含树中下一个fiber节点的引用或者为`null`。在这种情况下（译注：`nextUnitOfWork=null`的情况）React退出work循环并准备提交变化。

有四个主要函数用于遍历树，开始或结束work：
- [performUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1056)
- [beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489)
- [completeUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L879)
- [completeWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberCompleteWork.js#L532)

为了演示它们是如何使用的，看看下面遍历fiber树的动画。我在demo中使用这些函数的简化实现。每个函数都接收一个fiber节点来处理，随着React向下遍历树你会看到当前活动的fiber节点发生了变化。你可以在视频中清楚地看出算法是如何从一个分支到其他分支的。它在移动到父节点之前先完成子节点的work。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb197794ba7a4a32b339af08195f83cd~tplv-k3u1fbpfcp-zoom-1.image)

> 注意，垂直向下连着的表示兄弟节点，弯着连接的表示子节点，例如`b1`没有子节点，而`b2`有一个`c1`子节点

[这是视频的链接](https://vimeo.com/302222454)，你可以暂停播放，查看当前节点和函数的状态。从概念上讲，你可以把”开始“看作”进入“一个组件，把”完成“看作“退出它”。当我说明这些函数是做说明的时候，你可以[在这查看示例和实现](https://stackblitz.com/edit/js-ntqfil?file=index.js)。

让我们从前面的两个函数`performUnitOfWork`和`beginWork`开始：

```js
function performUnitOfWork(workInProgress) {
    let next = beginWork(workInProgress);
    if (next === null) {
        next = completeUnitOfWork(workInProgress);
    }
    return next;
}

function beginWork(workInProgress) {
    console.log('work performed for ' + workInProgress.name);
    return workInProgress.child;
}
```
`performUnitOfWork`函数接收一个`workInProgress`树中的fiber节点，通过调用`beginWork`函数开始工作。这个函数将开启一个fiber节点所有执行的活动。为了演示，我们简单的打印出fiber的name表示work已经完成。**`beginWork`函数总是返回一个指向循环中下一个待处理child的指针或者`null`**。

如果存在下一个child，它将在`workLoop`函数中赋值给`nextUnitOfWork`变量。 但是，如果不存在child，React知道它已经到达分支的末尾，所以它可以完成当前节点。**一旦节点完成，它需要执行兄弟节点的work然后返回父节点**。这是在`completeUnitOfWork`函数内完成的:

```js
function completeUnitOfWork(workInProgress) {
    while (true) {
        let returnFiber = workInProgress.return;
        let siblingFiber = workInProgress.sibling;

        nextUnitOfWork = completeWork(workInProgress);

        if (siblingFiber !== null) {
            // If there is a sibling, return it
            // to perform work for this sibling
            return siblingFiber;
        } else if (returnFiber !== null) {
            // If there's no more work in this returnFiber,
            // continue the loop to complete the parent.
            workInProgress = returnFiber;
            continue;
        } else {
            // We've reached the root.
            return null;
        }
    }
}
function completeWork(workInProgress) {
    console.log('work completed for ' + workInProgress.name);
    return null;
}
```

你可以看出这个函数主体就是一个大的`while`循环。当一个`workInProgress`节点没有子节点时React会进入这个函数。在当前fiber完成work后，会检查是否有兄弟节点。如果有，React退出这个函数并返回指向兄弟节点的指针。它将赋值给`nextUnitOfWork`变量，React将从这个兄弟节点开始为这个分支执行work。在这个时候，React只完成了之前兄弟节点的work，理解这点很重要。它没有完成父节点的work。**只有当所有以子节点开始的分支都完成后，它才完成父节点的work并回到父节点。**

你可以从实现中看出，`performUnitOfWork`和`completeUnitOfWork`主要起到迭代的作用，而主要活动是在`beginWork`和`completeWork`函数中进行的。在这个系列接下来的文章中，我们将学到当React进入`beginWork`和`completeWork`函数中时`ClickCounter`组件和`span`节点发生了什么。

## Commit阶段
这个阶段开始于[completeRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L2306)函数.在这个阶段中React更新DOM，调用变更前后生命周期函数。

当React进入这个阶段，它有两颗树和effects链表。一颗树代表当前渲染在屏幕上的状态。然后有颗在`render`阶段创建的`alternate`树。在源码中它被称为`finishedWork`或`workInProgress`，代表需要被显示到屏幕上的状态。`alternate`树和`current`树类似，通过child和sibling指针连接。

然后，还有一条effects链 —— `finishedWork`树节点的子集，通过`nextEffect`指针连接的。记住effect链是`render`阶段的运行结果。render阶段的目标就是确定哪些节点需要插入、更新或删除 ，以及需要调用哪些组件的生命周期方法。**这就是在commit阶段遍历的节点集。**

> 为了调试，`current`树可通过`fiber root`的`current`属性访问。`finishedWork`树可以通过`current`树中`HostFiber`节点的`alternate`属性访问。
`commit`阶段的主要函数是`commitRoot`。 大体上，它做了下面这些事：

- 在带有`Snapshot`effect标记的节点上调用 `getSnapshotBeforeUpdate`生命周期方法
- 在带有`Deletion`effect标记的节点上调用 `componentWillUnmount`生命周期方法
- 执行所有DOM的插入、更新、删除
- 设置`finishedWork`树作为`current`树
- 在带有`Placement`effect标记的节点上调用 `componentDidMount`生命周期方法
- 在带有`Update`effect标记的节点上调用 `componentDidUpdate`生命周期方法

在调用变更前方法`getSnapshotBeforeUpdate`之后，React在树中提交所有副作用。分成两次完成。第一次执行所有DOM（host）的插入、更新、删除，和ref卸载。然后，React将`finishedWork`树分配给`FiberRoot`，标记`workInProgress`树作为`current`树。这是在commit阶段第一部分之后，第二个部分之前完成的。所以在`componentWillUnmount`中之前的树仍然是当前的。`componentDidMount/Update`中`finished`树是当前的。在第二部分中React调用其他生命周期方法和ref回调。这些方法作为独立部分执行，因此所有的插入、更新和删除在整颗树中都已被调用。

这是运行上述步骤的函数的大体结构：
```js
function commitRoot(root, finishedWork) {
    commitBeforeMutationLifecycles()
    commitAllHostEffects();
    root.current = finishedWork;
    commitAllLifeCycles();
}
```
每个子函数都实现循环遍历effects list并检查effects的类别。当发现effect和该函数作用有关时会应用它。

### 变更前生命周期方法
例如，这是遍历effects树并检查节点是否有`Snapshot`effect的代码：
```js
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
对于类组件，effect意味着调用`getSnapshotBeforeUpdate`生命周期方法。
### DOM更新
[commitAllHostEffects](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L376)是React执行更新DOM的函数。这个函数大体上定义了对节点要做的操作类型并执行它：
```js
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
有趣的是，React在`commitDeletion`函数中调用`componentWillUnmount`方法作为删除过程的一部分。

### 变更后生命周期方法
[commitAllLifecycles](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L465)是React调用剩下的`componentDidUpdate`和`componentDidMount`生命周期方法的函数。

终于结束了。在评论区中告诉我你觉得这篇文章怎么样或问我问题。查看这个系列的下一篇文章[深入理解React中的state和props更新](https://indepth.dev/in-depth-explanation-of-state-and-props-update-in-react/)。我计划写更多的文章深入解释调度器，协调过程，以及effects list是如何创建的。我也计划创建个视频，使用这篇文章作基础展示如何调试程序。