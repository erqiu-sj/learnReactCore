# Fiber

`React` 中践行的就是代数效应,`hooks`就一个很好的例子

> `代数效应`就能将`副作用`从函数逻辑中抽离出来，使函数的关注点更纯粹

> 我们不需要关注`FunctionComponent`的`state`在`hooks`中是如何保存的，我们只需要编写业务逻辑就行

## 代数效应和 Generatro

从`React15`-`React16`，Reconciler 重构的一大目的就是：将老的`同步更新`架构变为`异步可中断更新`

`异步可中断`可以理解为：`更新`在执行过程中随时都可以被打断(浏览器时间切片或者有更高优先级的任务插队),当可以继续执行的时候就会恢复之前执行的中间状态

其实原生就已实现`generator`

- 类似`async`，`generator`也是具有传染性的，使用了`generator`则上下文的其他函数也需要做出改变，这样的心智负担比较重

- `generator`执行的中间状态的上下文关联的

```javascript
function* doWork(A, B, C) {
  var x = doExpensiveWorkA(A);
  yield;
  var y = x + doExpensiveWorkB(B);
  yield;
  var z = y + doExpensiveWorkC(C);
  return z;
}
```

> 当浏览器的空闲时间会依次执行，当浏览器时间切片用尽，则会在此恢复从中断位置继续执行，只考虑`单一优先级的中断与继续`情况下，`generator`可以很好的实现，但我们需要考虑`高优先级任务插队的情况下`，计算出 `x` and `y` 时，此时有个组件接收到一个高优先更新，则会中断，再继续执行时 `y` 就不能复用之前的状态，则需要重新计算，通过全局变量又会引入新的负责度

#### 可以理解为 `React` 在内部实现的一套状态更新机制，支持任务不同的优先级，可中断与恢复，并且恢复之后可以复用之前的中间状态，每一个任务更新单元为`React Element`对应的`Fiber`节点

# fiber 起源

> 为了解决`React15`中，`Reconciler`采用`递归创建虚拟dom`，递归过程不能中断，层级深时，会造成线程卡顿，`React16`将递归无法中断更新，改为了`异步可中断更新`由于旧的`虚拟dom`的数据结构无法满足，所以新的 fiber 就应运而生

## fiber 的三层含义

- 架构：之前 `React15` 的 `Reconciler`采用递归的方式执行，数据保存在递归调用栈中，所以被称为 `stack Reconciler`。`React16` 的 `Reconciler` 基于 Fiber 节点实现，被称为 `Fiber Reconciler`

- 静态数据结构：对应`React Element`,保存了该组件的类型(fnComponents\classComponeent\hostComponent)，对应的 dom 节点等信息

- 动态的工作单元来说：每一个`fiber`节点都保存了本次更新中该组件变化的状态，以及需要执行的动作,也就是`crud tag`

# 架构

```javascript
// 多个fiber节点是如何连接成树的呢？考fiber中的三个属性
// 指向父fiber
this.return = null;
// 指向子fiber
this.child = null;
// 执行兄弟fiber
this.sibling = null;
```

> 这里需要提一下，为什么父级指针叫做 return 而不是 parent 或者 father 呢？因为作为一个工作单元，return 指节点执行完 completeWork（本章后面会介绍）后会返回的下一个节点。子 Fiber 节点及其兄弟节点完成工作后会返回其父级节点，所以用 return 指代父级节点。

# 作为静态数据结构，保存了组件相关信息

```javascript
// Fiber对应组件的类型 Function/Class/Host...
this.tag = tag;
// key属性
this.key = key;
// 大部分情况同type，某些情况不同，比如FunctionComponent使用React.memo包裹
this.elementType = null;
// 对于 FunctionComponent，指函数本身，对于ClassComponent，指class，对于HostComponent，指DOM节点tagName
this.type = null;
// Fiber对应的真实DOM节点
this.stateNode = null;
```

# 作为动态工作单元

```javascript
// 保存本次更新造成的状态改变相关信息
this.pendingProps = pendingProps;
this.memoizedProps = null;
this.updateQueue = null;
this.memoizedState = null;
this.dependencies = null;

this.mode = mode;

// 保存本次更新会造成的DOM操作
this.effectTag = NoEffect;
this.nextEffect = null;

this.firstEffect = null;
this.lastEffect = null;
// 调度优先级相关
this.lanes = NoLanes;
this.childLanes = NoLanes;
```

# fiber 工作原理

## 双缓存

> 指在内存中构建并直接替换的技术

**_好处：不会出现从白屏到出现画面的闪烁情况_**

## 双缓存 `fiber` 树

最多会出现两颗`fiber`树，页面显示的叫`current fiber树`，内存中构建的`workInProgress fiber`树

```javascript
currentFiber.alternate === workInProgressFiber;

workInProgressFiber.alternate === currentFiber;
```

`React`应用的节点根通过`current`指针在不同的`fiber`树的`rootfiber(AppComponent)`间来回切换实现`fiber`树的切换

每次状态更新都会产生新的`workInProgress`树，通`curent`与`workInProgress`的替换，来完成`dom`更新

# mount 时(构建)

- 首次执行`React.render`创建`fiberRootNode(root)`and`rootFiber(App)`

```javascript
fiberRootNode.current = rootFiber;
```

> 首屏渲染没有挂载任何`dom`所以，指向的`rootFiber`没有子节点，`current fiber`树为`null`

- `render`阶段，根据组件返回的`jsx`在内存依次创建`fiber节点`
  并连接在一起构建`fiber`树
  ，在构建中`workInProgress`树会复用`current fiber`树中已有的`fiber`节点内的属性，当`workInProgress fiber`树在内存中创建完成的时候

```javascript
rootFiber.alternate = workInProgress.alternate;
```

> 然后再`commit阶段`渲染到页面

# update 时

- 我们点击某一节点触发更新状态改变时，会开启依次新的`render阶段`，并构建一颗心的`workInProgress fiber`树，和`mount`一样，`workInProgress fiber`的创建也会复用`current fiber`树对应的节点数据(diff 算法)

- 完成构建后会进入`commit`阶段渲染到页面上，渲染完毕后`workInProgress fiber`树变成`current fiber`树
