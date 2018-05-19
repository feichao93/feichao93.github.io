# Building Your Own Redux-Saga

https://zhuanlan.zhihu.com/p/35437092

本文涉及的 redux-saga 源码都来自 [redux-saga v1.0.0-beta.1](https://github.com/redux-saga/redux-saga/tree/v1.0.0-beta.1) 版本。

本文实现的 little-saga 对 redux-saga 进行了很多简化，主要有很多实现细节都是不一样的。

(todo) 将 iter 重命名为 iterator; effect-producer; effect-runner; effects 消费者直接使用 redux-saga 时，redux-saga 是消费者，而我们的代码一直充当着 effects 生产者的角色。

很多时候也可以将 redux-saga 看作一种「请求-响应」模型，我们业务代码生产 effects 以发起「请求」，而 redux-saga 负责消费这些 effects 并将响应返回给业务代码。

## 1.1 同步 简单的生产者和 for-of 消费者

```javascript
function* range(start, end) {
  for (let i = start; i < end; i++) {
    yield i
  }
}

for (let x of range(1, 10)) {
  console.log(x)
}
// 输出 1, 2, 3 ... 8, 9
```

(todo) for-of 循环做了什么？ 什么是迭代器？ 什么是生成器？

http://www.infoq.com/cn/articles/es6-in-depth-iterators-and-the-for-of-loop

http://www.infoq.com/cn/articles/es6-in-depth-generators?utm_source=articles_about_ES6-In-Depth&utm_medium=link&utm_campaign=ES6-In-Depth

## 1.2 同步 简单的生产者 与 while-true 消费者

如果我们不用 for-of，而是自己实现消费者，那么代码如下：

```javascript
const iter = range(1, 10)
while (true) {
  const { done, value } = iter.next()
  if (done) {
    break
  }
  console.log(value)
}
// 输出 1, 2, 3 ... 8, 9
```

(todo) 代码执行的顺序的说明

## 1.3 异步 简单生产者 与 消费者

每次调用 iter.next() 都可以从生产者中取回一个结果；上面的例子中我们用了 while-true 循环，所有的代码都是同步的；这一次我们利用 setTimeout 使得前后两次调用 iter.next() 之间有一定的间隔。为了实现在不同时刻调用 iter.next()，我们也将驱动函数 next 写成了递归的形式。(todo) 这里是否需要将 `arg` 写出来？

```javascript
const iter = range(1, 10)

function next(arg) {
  const { done, value } = iter.next(arg)
  if (done) {
    return
  }
  console.log(value)
  setTimeout(next, value * 100)
}
next()
// 陆续输出 1, 2, 3 ... 8, 9
// 前后两个数字输出之间有一定的时间间隔，且数字越大输出间隔越长
```

## 1.4 双向通信

在 redux-saga 中，我们往往需要使用 yield 语句的返回值，返回值的含义取决于 effect 的类型。例如下面这个例子（注意这个例子需要依赖于 redux-saga，不能像其他例子那样直接运行在浏览器中）：

```javascript
function* generatorInReduxSaga() {
  const response = yield fetch('https://example.com/')
  const action = yield take('SOME_ACTION')
  const raceResult = yield race({ a: effect1, b: effect2 })
}
```

为了实现双向通信，effect-runner 要提供合适的参数来调用 `iter.next(arg)` 。当 `iter.next(arg)` 被调用是，参数 arg 将会作为 `yield xxx` 语句的返回值，且暂停的迭代器会继续执行（直到遇到下一个 yield 语句）。为此我们修改前面的代码如下：

```javascript
const iter = range(1, 10)

function next(arg) {
  // 注意驱动函数多了参数 arg
  // 这里我们将 arg 作为参数传递给 iter.next，作为 effect-producer 中 yield 语句的返回值
  const { done, value } = iter.next(arg)
  if (done) {
    return
  }
  console.log(value)
  // 在这个简单的例子中，我们的 effect-runner 忽略了 yield 语句的返回值
  // 所以 ??? 处的参数现在似乎还没什么意义……
  setTimeout(() => next(/* ??? */), value * 100)
}
next()
// 陆续输出 1, 2, 3 ... 8, 9
// 前后两个数字输出之间有一定的时间间隔，且数字越大输出间隔越长
```

## 1.5 effect type

前面的例子中我们的 effect-producer 都是简单的 range，effect（即被 yield 的值）为数字。因为数字没有什么确切的含义，effect-runner 只是简单地打印这些数字，然后再在合适地时刻调用 `iterator.next(arg)`。

如果 effect 有明确的含义，effect-runner 就可以根据其含义来决定具体的执行逻辑。redux-saga 可以处理 promise、iterator、take、put 等类型的 effect，合理地组合不同类型的 effect 可以表达非常复杂的异步逻辑（例如 [坦克大战](https://zhuanlan.zhihu.com/p/35551654) 的游戏逻辑）。下面我们给 little-saga 加上一些简单的 effect 的处理能力。

```javascript
function* gen() {
  console.log('enter ...')
  const a = yield ['promise', fetch('/')]
  console.assert(a instanceof Response)
  const b = yield ['delay', 500]
  console.assert(b === '500ms elapsed')
  const c = yield ['ping']
  console.assert(c === 'pong')
  console.log('exit ... ')
}

const iterator = gen()

function next(arg, isErr) {
  // 注意驱动函数多了参数 isErr，用来表示是否发生了错误
  const { done, value } = iterator.next(arg)
  if (done) {
    return
  }
  // console.log(value) 不打印 value，而是根据 value 的含义执行相应的处理逻辑
  if (value[0] === 'promise') {
    const promise = value[1]
    promise.then(resolvedValue => next(resolvedValue), error => next(error, true))
  } else if (value[0] === 'delay') {
    const timeout = value[1]
    setTimeout(() => next(`${timeout}ms elapsed`), timeout)
  } else if (value[0] === 'ping') {
    next('pong')
  } else {
    iter.throw(new Error('无法识别的 effect'))
  }
}
next() // kick start!
```

## 1.6 proc 初步实现

(todo)

## 2.1 Task

[proc函数](https://github.com/redux-saga/redux-saga/blob/v1.0.0-beta.1/packages/core/src/internal/proc.js#L173) 用于运行一个迭代器，并返回一个 Task 对象。Task 对象描述了该迭代器的运行状态，我们首先来看看 Task 的接口（使用 TypeScript 来表示类型信息）。在 little-saga中，我们将使用相同的 Task 接口。

```typescript
type Callback = (result: any, isErr: boolean) => void
type Joiner = { task: Task; cb: Callback }

interface Task {
  [TASK]: true
  toPromise(): Promise<any>
  cont: Callback
  joiners: Joiner[]
	cancel(): void
  isRunning(): boolean
  isCancelled(): boolean
  isAborted(): boolean
  result(): any
  error(): Error
}
```

迭代器/saga实例在运行时，内部逻辑也许非常复杂，不过从外界看来，迭代器最终只返回一个异步结果。仅从返回值的角度看，我们用一个 Promise 就可以表示saga实例。Task 对象包含了 `toPromise()` 方法，该方法会返回saga实例对应的promise。在其他方面，saga实例的功能是要比Promise丰富的：`Task#cancel()` 方法使得saga实例允许被取消；`Task#isXXX()` 等方法可以查询saga实例的运行状态；`Task#result()` / `Task#error()` 方法可以获得saga实例的结果。

## 2.2 fork model

redux-saga 提供了 fork effect 来进行非阻塞调用，`yield fork(...)` 会返回一个 Task 对象，用于表示在后台执行的 saga 实例。在更普遍的情况下，一个 saga 实例在运行的时候会多次 yield fork effect，那么一个 parent-saga 实例就会有多个child-saga。rootSaga 通过 `sagaMiddleware.run()` 开始运行，在 rootSaga 运行过程中，会 fork 得到若干个 child-saga，每一个 child-saga 又会 fork 得到若干个 grandchild-saga，如果我们将所有的 parent-child 关系绘制出来的话，我们可以得到类似于下图这样的一棵 saga 树。

![saga-tree-of-running-battle-city](saga-tree-of-running-battle-city.jpg)

redux-saga 的文档也[对 fork model 进行了详细的说明](https://redux-saga.js.org/docs/advanced/ForkModel.html)，下面我做一点简单的翻译：

* 完成：一个 saga 实例在满足以下条件之后进入完成状态:
  1. 迭代器自身的语句执行完成
  2. 所有的 child-saga 进入完成状态
* 错误传播：一个 saga 实例在以下情况会中断并抛出错误：
* 1. 迭代器自身执行时抛出了异常
  2. 其中一个 child-saga 抛出了错误
* 取消：取消一个 saga 实例也会导致以下事情的发生：
* 1. 取消 mainTask，也就是取消当前 saga 实例等待的 effect
  2. 取消所有仍在执行的 child-saga

我们用一些坦克大战的例子来说明上面三个规则：

* 完成：当 gameSaga 迭代器自身的代码执行完毕，并且 tickEmitter/AIMaster/humanPlayer 这三个 child-saga 完成时，gameSaga 进入完成状态。而 AIMaster 的完成也意味着 addAIHandler 的完成，addAIHandler 的完成意味着所有的 AIPlayer 的完成。**当一个节点的所有子节点完成时，且自身迭代器代码执行完毕时，该节点才算完成。**
* 错误传播：假设 AIPlayer 发生错误，那么 addAIHandler 会收到该错误；假设 addAIHandler 无法处理该错误，该错误就会传播给 AIMaster。**当一个节点发生错误时，错误会沿着树向根节点向上传播，直到某个某个节点捕获该错误。**
* 取消：gameSaga 被取消时，tickEmitter/AIMaster/humanPlayer 会同时被取消，而取消 AIMaster 被取消时，AIHandler 会被同时取消。**取消一个节点时，该节点对应的整个子树都将被取消。**

## 2.3 fork-queue

fork-queue 是 fork model 的具体实现。redux-saga 使用了 [forkQueue](https://github.com/redux-saga/redux-saga/blob/v1.0.0-beta.1/packages/core/src/internal/proc.js#L73) 来实现，在 little-saga 中我们将使用同样的做法。

每一个 saga 实例可以用一个 Task 对象进行描述，为了实现 fork model，每一个 saga 实例开始运行时，我们需要用一个数组来保存 child-tasks。我们来看看 forkQueue 的接口：

```typescript
interface intefaceOfFunctionForkQueue {
  (mainTask: MainTask, cb: Callback): {
    addTask(task: Task): void
    cancelAll(): void
    abort(err: Error): void
  }
}
```

函数 `forkQueue` 接受两个参数，参数 `mainTask` 代表当前迭代器自身代码的执行状态，参数 `cb` 是整个 fork-queue 的回调函数。当所有的 child-task 以及 mainTask 都完成时，我们需要调用 `cb` 来通知其 parent-saga（对应于 *2.2 fork model中的「完成」*）。

函数 `forkQueue` 返回的对象包含三个方法。方法 `addTask` 用于向 fork-queue 中添加新的 child-task；方法 `cancelAll` 用于取消所有的 child-task；而方法 `abort` 不仅会取消所有的 child-task，还会调用 `cb` 向 parent-saga 通知错误。

```javascript
function forkQueue(mainTask, cb) {
  let tasks = []
  let result
  let completed = false

  addTask(mainTask)

  function abort(err) {
    cancelAll()
    cb(err, true)
  }

  function addTask(task) {
    tasks.push(task)
    task.cont = (res, isErr) => {
      if (completed) {
        return
      }

      remove(tasks, task)
      task.cont = noop
      if (isErr) {
        abort(res)
      } else {
        if (task === mainTask) {
          result = res
        }
        if (tasks.length === 0) {
          completed = true
          cb(result)
        }
      }
    }
  }

  function cancelAll() {
    if (completed) {
      return
    }
    completed = true
    tasks.forEach(t => {
      t.cont = noop
      t.cancel()
    })
    tasks = []
  }

  return {
    addTask,
    cancelAll,
    abort,
  }
}
```



saga 实例运行时可以动态地 fork 新的 child

例如应用执行了 `yield cancel(task1)`，那么 task1 的就需要对其

(todo) 需要讲清楚 模型fork-model 和 实现fork-queue 之间的关系。