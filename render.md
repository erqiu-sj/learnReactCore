# Render

`fiber节点`是如何被创建并构建`fiber树`？

`render阶段`开始于`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`方法的调用，取决于本次更新是同步更新还是异步更新

```javascript
// performSyncWorkOnRoot会调用该方法
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// performConcurrentWorkOnRoot会调用该方法
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

> 唯一的区别是是否调用`shouldYield`，如果当前浏览器帧没有剩余时间，`shouldYield`会终止循环，直到浏览器有空闲时间再继续遍历。

> `workInProgress`代表当前已创建的`workInProgress fiber`。

> `performUnitOfWork`方法会创建下一个`fiber 节点`并赋值给`workInProgress`，并将`workInProgress`与已创建的`fiber 节点`连接起来构建成为`fiber 树`

> 我们知道`fiber reconfiler`是从`stack reconciler`重构而来，通过遍历的方式实现可中断的递归，所以`performUintOfWork`的工作可以分为`递`和`归`

# "递"阶段

首先从`rootFiber`开始向下深度优先遍历，为遍历到每一个`fiber 节点`调用`beginWork`方法

This is function 会将传入的`fiber 节点`创建`子 fiber 节点`,并将这两个`fiber 节点`连接起来

当遍历到叶子节点,(即没有子组件的组件)时就会进入“归”的阶段

# "归"阶段

在“归”阶段会调用`completeWork`处理`fiber 节点`，当某个`fiber 节点`执行完`completeWork`方法，如果其存在`兄弟fiber`即(`fiber.sibling!==null`),会进入其`兄弟fiber`的`递`阶段，如果不存在，会进入`父fiber`的归阶段。

> ‘递’和‘归’阶段都会交错执行，直到`归`到`rootFiber`，`render 阶段结束`

```javascript
function App() {
  return (
    <div>
      i am
      <span>KaSong</span>
    </div>
  );
}

ReactDOM.render(<App />, document.getElementById("root"));
```

```javascript
1. rootFiber beginWork
2. App Fiber beginWork
3. div Fiber beginWork
4. "i am" Fiber beginWork
5. "i am" Fiber completeWork
6. span Fiber beginWork
7. span Fiber completeWork
8. div Fiber completeWork
9. App Fiber completeWork
10. rootFiber completeWork
```

> 注意

> 之所以没有 “KaSong” Fiber 的 beginWork/completeWork，是因为作为一种性能优化手段，针对只有单一文本子节点的 Fiber，React 会特殊处理。
