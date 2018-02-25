# 实用「撤销重做」指南

为一个应用实现撤销重做功能是一件不容易的事情。

<!-- 传统的 MVC 框架中，所有的模型都是可变的，为了实现撤销功能，我们必须在每一次改变状态之前记录原先的状态，这大大增加了工作量。如果使用 Redux 的话， [Redux](https://redux.js.org/recipes/implementing-undo-history) 中实现 -->

假装有个例子：网页 SLIDES 软件 （类似于 Microsoft Office PowerPoint）

### 第一步：确定哪些状态需要历史记录

并非所有的状态都需要历史记录。许多状态是非常琐碎的，尤其是一些与鼠标或者键盘交互相关的状态，例如在 PPT 中拖拽文本框时应用可能设置了 dragging 标记，应用视图模块会根据该标记显示对应的提示，该 dragging 标记不应当记录在历史记录中；而另一些状态无法被撤销（或是即使撤销也没什么意义），例如网页窗口大小，向后台发送过的请求列表等。

我们用 Immutable Record 封装需要历史记录的状态：

```typescript
// State.ts
import { Record, List, Set } from 'immutable'

/* other imports */
const StateRecord = Record({
  items: List<Item>
  transform: d3.ZoomTransform
  selection: number
})

// 用类封装，便于书写 TypeScript，注意这里要使用Immutable 4.0 以上的版本
export default class State extends StateRecord {}
```

### 第A步：为每种不同的操作创建对应的 Action 子类

与 redux-undo 不同的是，我们仍然采用命令模式：定义基类 Action，用 Action 实例封装对 State 的操作，并为不同的操作类型定义相应的 Action 子类。在 TypeScript 中，Action 基类用 Abstract Class 来定义比较方便。

```typescript
// actions/index.ts
export default abstract class Action {
  abstract next(state: State): State
  abstract prev(state: State): State

  prepare(appHistory: AppHistory): AppHistory { return appHistory }
  getMessage() { return this.constructor.name }
}
```

Action 对象的 next 方法用来计算「下一个状态」，prev 方法用来计算上一个状态。应用运行时，用户交互产生一个 Action 流，每次产生 Action 对象时，我们调用该对象的 next 方法，然后将其放入一个 action list 中；用户进行撤销操作时，我们从 action list 中取出最近一个 Action 并调用其 prev 方法。应用运行时，next/prev 方法被调用的顺序大致如下：

```typescript
// initState 是一开始就给定的应用初始状态
// 某一时刻，用户交互产生了 action1 ...
state1 = action1.next(initState)
// 同样的，又一个时刻，用户交互产生了 action2 ...
state2 = action2.next(state1)
state3 = action3.next(state2)
// 用户进行撤销，此时我们需要调用最近一个action的prev方法
state4 = action3.prev(state3)
// 如果再次进行撤销，我们从 action list 中拿出最近一个action，调用其prev方法
state5 = action2.prev(state4)
```

getMessage 方法用来获取 Action 对象的简短描述。通过该方法，我们可以将用户的操作记录显示在页面上，让用户更方便地了解最近进行的操作。prepare 方法用来在 Action 第一次被应用之前，使其「准备好」，AppHistory 的定义在本文后面会给出。

##### Applied-Action

为了方便后面的说明，我们对 Applied-Action 进行一个简单的定义：Applied-Action 是指那些已经反映在当前应用状态中的 action；当 action 的 next 方法执行时，该 action 变为 applied，当 prev 方法被执行时，该 action 变为unapplied。

##### Action 子类举例

下面是一个典型的 Action 子类，用于表达添加一个新的 Item 的操作。

```typescript
// actions/AddItemAction.ts
export default class AddItemAction extends Action {
  private newItem: Item
  private prevSelection: number

  constructor(newItem: Item) {
    super()
    this.newItem = newItem
  }
    
  prepare(history: AppHistory) {
    this.prevSelection = history.state.selection
    return history
  }

  next(state: State) {
    return state
      .setIn(['items', this.newItem.id], this.newItem)
      .set('selection', this.newItemId)
  }
  
  prev(state: State) {
    return state
      .deleteIn(['items', this.newItem.id])
      .set('selection', this.prevSelection)
  }
    
  getMessage() { return `Add item ${this.newItem.id}` }
}
```

创建新的 Item 之后会自动选中该 Item，为了使得撤销该操作时 state.selection 变为原来的值，我们在 prepare 方法中读取了「创建 Item 之前的 selection 的值」并保存在 prevSelection 字段中。

## 第B步：创建历史记录容器对象

前面的 `State` 类用于表示某个时刻应用的状态，接下来我们定义 `AppHistory` 类用来表示应用的历史记录。同样的，我们仍然使用 Immutable Record 来定义历史记录。其中 state 字段用来表达当前的应用状态，list 字段用来存放所有的 action，而 index 字段用来记录最近的 applied-action 的下标。应用的历史状态可以通过的 undo/redo 方法计算得到。具体代码如下：

```typescript
// AppHistory.ts
const emptyAction = Symbol('empty-action')
// TypeScript2.7之后对symbol的支持大大增强，我们在这里导出这些symbol
export const undo = Symbol('undo')
export const redo = Symbol('redo')

const AppHistoryRecord = Record({
  // 当前应用状态
  state: new State(),
  // action 列表
  list: List<Action>(),
  // index 表示最后一个applied-action在list中的下标。-1 表示没有任何applied-action
  index: -1,
})

export default class AppHistory extends AppHistoryRecord {
  pop() { // 移除最后一项操作记录
    return this
      .update('list', list => list.splice(this.index, 1))
      .update('index', x => x - 1)
  }
  getLastAction() { return this.index === -1 ? emptyAction : this.list.get(this.index) }
  getNextAction() { return this.list.get(this.index + 1, emptyAction) }

  apply(action: Action) {
    if (action === emptyAction) return this
    return this.merge({
      list: this.list.setSize(this.index + 1).push(action),
      index: this.index + 1,
      state: action.next(this.state),
    })
  }

  redo() {
    const action = this.getNextAction()
    if (action === emptyAction) return this
    return this.merge({
      list: this.list,
      index: this.index + 1,
      state: action.next(this.state),
    })
  }

  undo() {
    const action = this.getLastAction()
    if (action === emptyAction) return this
    return this.merge({
      list: this.list,
      index: this.index - 1,
      state: action.prev(this.state),
    })
  }
}
```

## 第C步：ALL TOGETHER

## TODO 合并 Action



