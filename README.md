**_该笔记仅为学习使用_**

**_该笔记所有内容都来自 (React 技术揭秘)[https://github.com/BetaSu/just-react]_**

# React 16

## 架构

- Scheduler (调度器)

  > 调度任务的优先级，高优任务会先进入 Reconciler

- Reconciler (协调器)

  > 负责找出变化的组件

- Renderer(渲染器)
  > 负责将变化的组件渲染到页面上

### Scheduler

`我们需要以浏览器是否有剩余时间作为任务中断标准，那么我们需要一种机制，当浏览器有剩余时间通知我们(回调)`

`为什么需要Scheduler，因为React15的架构为，同步递归渲染子组件，这意味着一旦递归开始，就无法中断，如果组件层级很深，一旦超过一针，就会造成页面的卡顿`

部分浏览器已经实现了这个 Api-`requestldleCallback`但是**浏览器兼容性差，触发频率不稳定，容易受到很多因素影戏，比如浏览器切换 tab 时**，所以`React`放弃使用

基于以上原因，React 实现了在空闲时间内触发回调的功能，还提供了多种调度优先级任务设置的功能，那就是 `Scheduler`

### Reconciler

`React15`中是递归处理虚拟 dom，而`React16`中变成了可中断的循环过程，每次循环都会调用`shouldYield(翻译:应该让步)`判断当前是否有剩余时间

```javascript
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

#### `React16` 如何解决中断更新时 Dom 渲染不完全的问题呢？

`React16`中`Reconciler`与`Renderer`不再是交替工作，当`Scheduler`将任务交给`Reconciler`时，`Reconciler`会为变化的`虚拟dom` 打上 `crud` 的 `tag`

整个`Scheduler`与`Reconciler`的工作都在内存中进行，只有当所有组件都完成了`Reconciler`的工作，才会统一的交给`Renderer`

### Renderer

`Renderer`会根据`Reconciler`为`虚拟Dom`打的`CRUD_Tag`,同步执行，对应的`Dom操作`

# 所以`React16`的更新过程为

`scheduler`和`reconciler`随时都有被打断的可能

- 在其他更高优任务需要先更新时
- 当前帧没有剩余时间

## 由于 Scheduler 和 Reconciler 都是在内存中进行的，所以即便反复中断，用户也不会看到更新不完全的 Dom
