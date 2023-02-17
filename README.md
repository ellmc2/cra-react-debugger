# cra-react-debugger

基于 cra 脚手架创建 react 项目并调试react@18.2.0源码

## 一、准备

```sh
# 1、初始化项目
npx create-react-app cra-debugger

# 2、使用cra提供的自定义配置
npm run eject

# 3、拉取v18.2.0 tag 的react
cd src && git clone https://github.com/facebook/react.git
# 更新分支并切换并创建18.2.0分支
git remote update && git checkout -b v18.2.0 v18.2.0
# 删除react文件夹下的.git文件夹

# 注意：需要切换node版本至v16，否则会报错：Current node version is not supported for development, expected "18.14.0" to satisfy "^12.17.0 || 13.x || 14.x || 15.x || 16.x || 17.x".
nvm use v16


# 4、打包react、scheduler、react-dom成cjs包
yarn build react/index,react/jsx,react-dom/index,scheduler --type=NODE

# 5、在源码目录 build/node_modules 为 react、react-dom 创建 yarn link
# 通过yarn link 可以改变项目中依赖包的目录指向
# 声明react指向
cd build/node_modules/react && yarn link
# 声明react-dom指向
cd build/node_modules/react-dom && yarn link

# 6、进入项目根目录，通过yarn link 将项目中的react、react-dom的指向刚刚打包好的react、react-dom
yarn link react react-dom

# 7、修改src/react/build/node_modules/react/cjs/react.development.js添加console.log，启动项目发现执行了打印

```

### 1、react 的目录结构

```
// src/react/packages
.
├── dom-event-testing-library
├── eslint-plugin-react-hooks
├── jest-mock-scheduler
├── jest-react
├── react
├── react-art // react-art 渲染器，用于渲染到canvas、svg
├── react-cache // 创建自定义的流
├── react-client // 创建自定义的流
├── react-debug-tools
├── react-devtools
├── react-devtools-core
├── react-devtools-extensions
├── react-devtools-inline
├── react-devtools-shared
├── react-devtools-shell
├── react-devtools-timeline
├── react-dom // react-dom 渲染器
├── react-fetch // 用于数据请求
├── react-fs
├── react-interactions // 用于测试交互相关的内部特性，比如React的事件模型
├── react-is // 用于测试组件是否是某类型
├── react-native-renderer // react-native 渲染器
├── react-noop-renderer // noop 渲染器，方便 debug fiber
├── react-pg
├── react-reconciler // 协调器
├── react-refresh // “热重载”的React官方实现
├── react-server // 创建自定义SSR流
├── react-server-dom-relay
├── react-server-dom-webpack
├── react-server-native-relay
├── react-suspense-test-utils
├── react-test-renderer
├── scheduler // 调度器
├── shared // 公共方法
├── use-subscription
└── use-sync-external-store
```

### 2、`render` 阶段

> `render阶段`开始于`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`方法的调用。这取决于本次更新是同步更新还是异步更新。我们现在还不需要学习这两个方法，只需要知道在这两个方法中会调用如下两个方法：

```js
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

> 可以看到，他们唯一的区别是是否调用`shouldYield`。如果当前浏览器帧没有剩余时间，`shouldYield`会中止循环，直到浏览器有空闲时间后再继续遍历。

> `workInProgress`代表当前已创建的`workInProgress fiber`。

> `performUnitOfWork`方法会创建下一个`Fiber节点`并赋值给`workInProgress`，并将`workInProgress`与已创建的`Fiber节点`连接起来构成`Fiber树`。

> 我们知道`Fiber Reconciler`是从`Stack Reconciler`重构而来，通过遍历的方式实现可中断的递归，所以`performUnitOfWork`的工作可以分为两部分：“递”和“归”。

#### 2.1 “递”阶段

> 首先从`rootFiber`开始向下深度优先遍历。为遍历到的每个`Fiber节点`调用[beginWork 方法 ](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3058)。

> 该方法会根据传入的`Fiber节点`创建`子Fiber节点`，并将这两个`Fiber节点`连接起来。

> 当遍历到叶子节点（即没有子组件的组件）时就会进入“归”阶段。

#### 2.2 “归”阶段

> 在“归”阶段会调用[completeWork](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L652)处理`Fiber节点`。

> 当某个`Fiber节点`执行完`completeWork`，如果其存在`兄弟Fiber节点`（即`fiber.sibling !== null`），会进入其`兄弟Fiber`的“递”阶段。

> 如果不存在`兄弟Fiber`，会进入`父级Fiber`的“归”阶段。

> “递”和“归”阶段会交错执行直到“归”到`rootFiber`。至此，`render阶段`的工作就结束了。

##### 2.2.1 例：简单介绍下下面代码的 render 阶段执行过程

```js
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

解：

对应 `fiber树` 结构：

 <img src="/Users/xuhao/Documents/GitHub/cra-react-debugger/assets/img/fiber.png" alt="fiber" style="zoom:25%;" />

`render阶段` 会依次执行：

```md
rootFiber beginWork
App Fiber beginWork
div Fiber beginWork
"i am" Fiber beginWork
"i am" Fiber completeWork
span Fiber beginWork
span Fiber completeWork
div Fiber completeWork
App Fiber completeWork
rootFiber completeWork
```

> 注意：之所以没有 “KaSong” Fiber 的 beginWork/completeWork，是因为作为一种性能优化手段，针对只有单一文本子节点的`Fiber`，`React`会特殊处理。

> 以上介绍了`render阶段`会调用的方法。在接下来会讲解`beginWork`和`completeWork`做的具体工作。

##### 2.2.2 beginWork

上一节我们了解到`render阶段`的工作可以分为“递”阶段和“归”阶段。其中“递”阶段会执行 `beginWork`，“归”阶段会执行`completeWork`。这一节我们看看“递”阶段的`beginWork`方法究竟做了什么。

###### 2.2.2.1 方法概览

> `beginWork` 的工作是传入 `当前Fiber节点`，创建 `子Fiber节点`，我们从传参来看看具体是如何做的。

```js
function beginWork(
  current: Fiber | null, // 当前组件对应的Fiber节点在上一次更新时的Fiber节点，即workInProgress.alternate
  workInProgress: Fiber, // 当前组件对应的Fiber节点
  renderLanes: Lanes // 优先级相关，在讲解Scheduler时再讲解
): Fiber | null {
  // ...省略函数体
}
```

> 从[双缓存机制一节](https://react.iamkasong.com/process/doubleBuffer.html)我们知道，除[`rootFiber`](https://react.iamkasong.com/process/doubleBuffer.html#mount时)以外， 组件 `mount` 时，由于是首次渲染，是不存在当前组件对应的 `Fiber节点` 在上一次更新时的 `Fiber节点`，即 `mount` 时 `current === null`。组件 `update` 时，由于之前已经 `mount` 过，所以 `current !== null`。

> 所以我们可以通过 `current === null ?` 来区分组件是处于 `mount` 还是 `update`。

> 基于此原因，`beginWork` 的工作可以分为两部分：

- `update` 时：如果 `current` 存在，在满足一定条件时可以复用 `current` 节点，这样就能克隆 `current.child` 作为 `workInProgress.child`，而不需要新建 `workInProgress.child`。
- `mount` 时：除 `fiberRootNode` 以外，`current === null`。会根据 `fiber.tag` 不同，创建不同类型的 `子Fiber节点`。

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  // update时：如果current存在可能存在优化路径，可以复用current（即上一次更新的Fiber节点）
  if (current !== null) {
    // ...省略

    // 复用current
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  } else {
    didReceiveUpdate = false;
  }

  // mount时：根据tag不同，创建不同的子Fiber节点
  switch (workInProgress.tag) {
    case IndeterminateComponent:
    // ...省略
    case LazyComponent:
    // ...省略
    case FunctionComponent:
    // ...省略
    case ClassComponent:
    // ...省略
    case HostRoot:
    // ...省略
    case HostComponent:
    // ...省略
    case HostText:
    // ...省略
    // ...省略其他类型
  }
}
```

###### 2.2.2.2 Update 时

> 我们可以看到，满足如下情况时 `didReceiveUpdate === false`（即可以直接复用前一次更新的 `子Fiber`，不需要新建 `子Fiber`）。

> 1. `oldProps === newProps && workInProgress.type === current.type`，即`props`与`fiber.type`不变
> 2. `!includesSomeLane(renderLanes, updateLanes)`，即当前`Fiber节点`优先级不够，会在讲解`Scheduler`时介绍

###### 2.2.2.3 Mount 时

当不满足优化路径时，我们就进入第二部分，新建 `子Fiber`。

我们可以看到，根据 `fiber.tag`不同，进入不同类型`Fiber`的创建逻辑。

```js
// mount时：根据tag不同，创建不同的Fiber节点
switch (workInProgress.tag) {
  case IndeterminateComponent:
  // ...省略
  case LazyComponent:
  // ...省略
  case FunctionComponent:
  // ...省略
  case ClassComponent:
  // ...省略
  case HostRoot:
  // ...省略
  case HostComponent:
  // ...省略
  case HostText:
  // ...省略
  // ...省略其他类型
}
```

对于我们常见的组件类型，如（`FunctionComponent`/`ClassComponent`/`HostComponent`），最终会进入[reconcileChildren](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L233)方法。

###### 2.2.2.4 reconcileChildren

从该函数名就能看出这是 `Reconciler` 模块的核心部分。那么他究竟做了什么呢？

- 对于 `mount` 的组件，他会创建新的 `子Fiber节点`；

- 对于 `update` 的组件，他会将当前组件与该组件在上次更新时对应的 `Fiber节点` 比较（也就是俗称的`Diff`算法），将比较的结果生成新 `Fiber节点`。

  ```js
  export function reconcileChildren(
    current: Fiber | null,
    workInProgress: Fiber,
    nextChildren: any,
    renderLanes: Lanes
  ) {
    if (current === null) {
      // 对于mount的组件
      workInProgress.child = mountChildFibers(
        workInProgress,
        null,
        nextChildren,
        renderLanes
      );
    } else {
      // 对于update的组件
      workInProgress.child = reconcileChildFibers(
        workInProgress,
        current.child,
        nextChildren,
        renderLanes
      );
    }
  }
  ```

  不论走哪个逻辑，最终他会生成新的 `子Fiber节点` 并赋值给 `workInProgress.child`，作为本次 `beginWork`[返回值 ](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1158)，并作为下次 `performUnitOfWork` 执行时 `workInProgress` 的[传参 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1702)。

  注意 ⚠️：值得一提的是，`mountChildFibers` 与 `reconcileChildFibers` 这两个方法的逻辑基本一致。唯一的区别是：`reconcileChildFibers` 会为生成的 `Fiber节点` 带上 `effectTag` 属性，而 `mountChildFibers` 不会。

###### 2.2.2.5 effectTag

我们知道，`render阶段` 的工作是在内存中进行，当工作结束后会通知 `Renderer` 需要执行的 `DOM` 操作。要执行 `DOM` 操作的具体类型就保存在 `fiber.effectTag` 中。

> 你可以从[这里](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactSideEffectTags.js)看到 `effectTag` 对应的 `DOM` 操作。

```js
// DOM需要插入到页面中
export const Placement = /*                */ 0b00000000000010;
// DOM需要更新
export const Update = /*                   */ 0b00000000000100;
// DOM需要插入到页面中并更新
export const PlacementAndUpdate = /*       */ 0b00000000000110;
// DOM需要删除
export const Deletion = /*                 */ 0b00000000001000;
```

> 通过二进制表示 `effectTag`，可以方便的使用位操作为 `fiber.effectTag` 赋值多个 `effect`。

那么，如果要通知 `Renderer` 将 `Fiber节点` 对应的 `DOM节点` 插入页面中，需要满足两个条件：

1. `fiber.stateNode` 存在，即 `Fiber节点` 中保存了对应的 `DOM节点`；
2. `(fiber.effectTag & Placement) !== 0`，即 `Fiber节点` 存在 `Placement effectTag`。

我们知道，`mount` 时，`fiber.stateNode === null`，且在 `reconcileChildren` 中调用的 `mountChildFibers` 不会为 `Fiber节点` 赋值 `effectTag`。那么首屏渲染如何完成呢？

针对第一个问题，`fiber.stateNode `会在 `completeWork` 中创建，我们会在下一节介绍。

第二个问题的答案十分巧妙：假设 `mountChildFibers` 也会赋值 `effectTag`，那么可以预见 `mount` 时整棵 `Fiber树` 所有节点都会有 `Placement effectTag`。那么 `commit阶段` 在执行 `DOM` 操作时每个节点都会执行一次插入操作，这样大量的 `DOM`操作是极低效的。

为了解决这个问题，在 `mount` 时只有 `rootFiber` 会赋值 `Placement effectTag`，在 `commit阶段` 只会执行一次插入操作。

beginWork 流程图：

![beginWork](assets/img/beginWork.png)

##### 2.2.3 complateWork

在流程概览一节，我们了解组件在 `render阶段` 会经历 `beginWork` 与 `completeWork`。

上一节我们讲解了组件执行 `beginWork `后会创建 `子Fiber节点`，节点上可能存在 `effectTag`。

这一节让我们看看 `completeWork` 会做什么工作。

###### 2.2.3.1 方法概览

类似 `beginWork`，`completeWork` 也是针对不同 `fiber.tag` 调用不同的处理逻辑。

```js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      return null;
    case ClassComponent: {
      // ...省略
      return null;
    }
    case HostRoot: {
      // ...省略
      updateHostContainer(workInProgress);
      return null;
    }
    case HostComponent: {
      // ...省略
      return null;
    }
  // ...省略
```

我们重点关注页面渲染所必须的 `HostComponent`（即原生 `DOM组件` 对应的 `Fiber节点`），其他类型 `Fiber` 的处理留在具体功能实现时讲解。

###### 2.2.3.2 处理 `HostCompnent`

和 `beginWork` 一样，我们根据 `current === null ?` 判断是 `mount` 还是 `update`。

同时针对 `HostComponent`，判断 `update` 时我们还需要考虑 `workInProgress.stateNode != null ?`（即该 `Fiber节点`是否存在对应的 `DOM节点`）。

```js
case HostComponent: {
  popHostContext(workInProgress);
  const rootContainerInstance = getRootHostContainer();
  const type = workInProgress.type;

  if (current !== null && workInProgress.stateNode != null) {
    // update的情况
    // ...省略
  } else {
    // mount的情况
    // ...省略
  }
  return null;
}
```

###### 2.2.3.3 `update` 时

当 `update` 时，`Fiber节点` 已经存在对应 `DOM节点`，所以不需要生成 `DOM节点`。需要做的主要是处理 `props`，比如：

- `onClick`、`onChange`等回调函数的注册
- 处理 `style prop`
- 处理 `DANGEROUSLY_SET_INNER_HTML prop`
- 处理 `children prop`

我们去掉一些当前不需要关注的功能（比如 `ref`）。可以看到最主要的逻辑是调用 `updateHostComponent` 方法。

```js
if (current !== null && workInProgress.stateNode != null) {
  // update的情况
  updateHostComponent(
    current,
    workInProgress,
    type,
    newProps,
    rootContainerInstance
  );
}
```

你可以从[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L225)看到 `updateHostComponent` 方法定义。

在 `updateHostComponent` 内部，被处理完的 `props` 会被赋值给 `workInProgress.updateQueue`，并最终会在 `commit阶段` 被渲染在页面上。

```js
workInProgress.updateQueue = (updatePayload: any);
```

其中 `updatePayload` 为数组形式，他的偶数索引的值为变化的 `prop key`，奇数索引的值为变化的 `prop value`。

> 具体渲染过程见 [mutation 阶段一节](https://react.iamkasong.com/renderer/mutation.html#hostcomponent-mutation)

###### 2.2.3.4 `mount` 时

同样，我们省略了不相关的逻辑。可以看到，`mount` 时的主要逻辑包括三个：

- 为 `Fiber节点` 生成对应的 `DOM节点`；
- 将子孙 `DOM节点` 插入刚生成的 `DOM节点` 中；
- 与 `update` 逻辑中的 `updateHostComponent` 类似的处理 `props` 的过程；

```js
// mount的情况

// ...省略服务端渲染相关逻辑

const currentHostContext = getHostContext();
// 为fiber创建对应DOM节点
const instance = createInstance(
  type,
  newProps,
  rootContainerInstance,
  currentHostContext,
  workInProgress
);
// 将子孙DOM节点插入刚生成的DOM节点中
appendAllChildren(instance, workInProgress, false, false);
// DOM节点赋值给fiber.stateNode
workInProgress.stateNode = instance;

// 与update逻辑中的updateHostComponent类似的处理props的过程
if (
  finalizeInitialChildren(
    instance,
    type,
    newProps,
    rootContainerInstance,
    currentHostContext
  )
) {
  markUpdate(workInProgress);
}
```

还记得[上一节](https://react.iamkasong.com/process/beginWork.html#effecttag)我们讲到：`mount` 时只会在 `rootFiber` 存在`Placement effectTag`。那么`commit阶段`是如何通过一次插入`DOM`操作（对应一个`Placement effectTag`）将整棵 `DOM树` 插入页面的呢？

原因就在于 `completeWork` 中的 `appendAllChildren` 方法。

由于 `completeWork` 属于“归”阶段调用的函数，每次调用 `appendAllChildren` 时都会将已生成的子孙 `DOM节点` 插入当前生成的 `DOM节点` 下。那么当“归”到 `rootFiber` 时，我们已经有一个构建好的离屏 `DOM树`。

###### 2.2.3.5 `effectList`

至此 `render阶段` 的绝大部分工作就完成了。

还有一个问题：作为 `DOM` 操作的依据，`commit阶段` 需要找到所有有 `effectTag` 的 `Fiber节点` 并依次执行 `effectTag` 对应操作。难道需要在 `commit阶段` 再遍历一次 `Fiber树` 寻找 `effectTag !== null` 的 `Fiber节点` 么？

这显然是很低效的。

为了解决这个问题，在 `completeWork` 的上层函数 `completeUnitOfWork` 中，每个执行完 `completeWork` 且存在 `effectTag` 的 `Fiber节点` 会被保存在一条被称为 `effectList` 的单向链表中。

`effectList ` 中第一个 `Fiber节点` 保存在 `fiber.firstEffect`，最后一个元素保存在 `fiber.lastEffect`。

类似 `appendAllChildren`，在“归”阶段，所有有 `effectTag` 的 `Fiber节点` 都会被追加在 `effectList` 中，最终形成一条以 `rootFiber.firstEffect` 为起点的单向链表。

```js
                       nextEffect         nextEffect
rootFiber.firstEffect -----------> fiber -----------> fiber
```

这样，在 `commit阶段` 只需要遍历 `effectList` 就能执行所有 `effect` 了。

你可以在[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1744)看到这段代码逻辑。

借用 `React` 团队成员**Dan Abramov**的话：`effectList ` 相较于 `Fiber树`，就像圣诞树上挂的那一串彩灯。

###### 2.2.3.6 流程结尾

至此，`render阶段` 全部工作完成。在 `performSyncWorkOnRoot` 函数中 `fiberRootNode` 被传递给 `commitRoot` 方法，开启 `commit阶段` 工作流程。

![](assets/img/completeWork.png)

### 3、`commit` 阶段

上一章[最后一节](https://react.iamkasong.com/process/completeWork.html#流程结尾)我们介绍了，`commitRoot` 方法是 `commit阶段` 工作的起点。`fiberRootNode `会作为传参。

```js
commitRoot(fiberRootNode);
```

在 `rootFiber.firstEffect` 上保存了一条需要执行 `副作用` 的 `Fiber节点` 的单向链表 `effectList`，这些 `Fiber节点` 的 `updateQueue` 中保存了变化的 `props`。

这些 `副作用` 对应的 `DOM操作` 在 `commit` 阶段执行。

除此之外，一些生命周期钩子（比如 `componentDidXXX` ）、`hook`（比如 `useEffect`）需要在 `commit` 阶段执行。

`commit` 阶段的主要工作（即 `Renderer` 的工作流程）分为三部分：

- before mutation 阶段（执行 `DOM` 操作前）
- mutation 阶段（执行 `DOM` 操作）
- layout 阶段（执行 `DOM` 操作后）

你可以从[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2001)看到`commit`阶段的完整代码

在 `before mutation阶段` 之前和 `layout阶段` 之后还有一些额外工作，涉及到比如 `useEffect` 的触发、`优先级相关` 的重置、`ref `的绑定/解绑。

这些对我们当前属于超纲内容，为了内容完整性，在这节简单介绍。

#### 3.1 流程概览

##### 3.1.1 `before mutation` 之前

`commitRootImpl `方法中直到第一句 `if (firstEffect !== null)` 之前属于 `before mutation` 之前。

我们大体看下他做的工作，现在你还不需要理解他们。

```js
do {
  // 触发useEffect回调与其他同步任务。由于这些任务可能触发新的渲染，所以这里要一直遍历执行直到没有任务
  flushPassiveEffects();
} while (rootWithPendingPassiveEffects !== null);

// root指 fiberRootNode
// root.finishedWork指当前应用的rootFiber
const finishedWork = root.finishedWork;

// 凡是变量名带lane的都是优先级相关
const lanes = root.finishedLanes;
if (finishedWork === null) {
  return null;
}
root.finishedWork = null;
root.finishedLanes = NoLanes;

// 重置Scheduler绑定的回调函数
root.callbackNode = null;
root.callbackId = NoLanes;

let remainingLanes = mergeLanes(finishedWork.lanes, finishedWork.childLanes);
// 重置优先级相关变量
markRootFinished(root, remainingLanes);

// 清除已完成的discrete updates，例如：用户鼠标点击触发的更新。
if (rootsWithPendingDiscreteUpdates !== null) {
  if (
    !hasDiscreteLanes(remainingLanes) &&
    rootsWithPendingDiscreteUpdates.has(root)
  ) {
    rootsWithPendingDiscreteUpdates.delete(root);
  }
}

// 重置全局变量
if (root === workInProgressRoot) {
  workInProgressRoot = null;
  workInProgress = null;
  workInProgressRootRenderLanes = NoLanes;
} else {
}

// 将effectList赋值给firstEffect
// 由于每个fiber的effectList只包含他的子孙节点
// 所以根节点如果有effectTag则不会被包含进来
// 所以这里将有effectTag的根节点插入到effectList尾部
// 这样才能保证有effect的fiber都在effectList中
let firstEffect;
if (finishedWork.effectTag > PerformedWork) {
  if (finishedWork.lastEffect !== null) {
    finishedWork.lastEffect.nextEffect = finishedWork;
    firstEffect = finishedWork.firstEffect;
  } else {
    firstEffect = finishedWork;
  }
} else {
  // 根节点没有effectTag
  firstEffect = finishedWork.firstEffect;
}
```

可以看到，`before mutation` 之前主要做一些变量赋值，状态重置的工作。

这一长串代码我们只需要关注最后赋值的 `firstEffect`，在 `commit` 的三个子阶段都会用到他。

##### 3.1.2 `layout` 之后

接下来让我们简单看下 `layout` 阶段执行完后的代码，现在你还不需要理解他们：

```js
const rootDidHavePassiveEffects = rootDoesHavePassiveEffects;

// useEffect相关
if (rootDoesHavePassiveEffects) {
  rootDoesHavePassiveEffects = false;
  rootWithPendingPassiveEffects = root;
  pendingPassiveEffectsLanes = lanes;
  pendingPassiveEffectsRenderPriority = renderPriorityLevel;
} else {
}

// 性能优化相关
if (remainingLanes !== NoLanes) {
  if (enableSchedulerTracing) {
    // ...
  }
} else {
  // ...
}

// 性能优化相关
if (enableSchedulerTracing) {
  if (!rootDidHavePassiveEffects) {
    // ...
  }
}

// ...检测无限循环的同步任务
if (remainingLanes === SyncLane) {
  // ...
}

// 在离开commitRoot函数前调用，触发一次新的调度，确保任何附加的任务被调度
ensureRootIsScheduled(root, now());

// ...处理未捕获错误及老版本遗留的边界问题

// 执行同步任务，这样同步任务不需要等到下次事件循环再执行
// 比如在 componentDidMount 中执行 setState 创建的更新会在这里被同步执行
// 或useLayoutEffect
flushSyncCallbackQueue();

return null;
```

你可以在[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2195)看到这段代码。

主要包括三点内容：

1. `useEffect` 相关的处理。我们会在讲解 `layout阶段` 时讲解。

2. 性能追踪相关。

   源码里有很多和 `interaction` 相关的变量。他们都和追踪 `React` 渲染时间、性能相关，在[Profiler API (opens new window)](https://zh-hans.reactjs.org/docs/profiler.html)和[DevTools (opens new window)](https://github.com/facebook/react-devtools/pull/1069)中使用。

   你可以在这里看到[interaction 的定义](https://gist.github.com/bvaughn/8de925562903afd2e7a12554adcdda16#overview)

3. 在`commit`阶段会触发一些生命周期钩子（如 `componentDidXXX`）和`hook`（如`useLayoutEffect`、`useEffect`）。

   在这些回调方法中可能触发新的更新，新的更新会开启新的 `render-commit` 流程。

######

## 参考

1. [React 技术揭秘](https://react.iamkasong.com/)
