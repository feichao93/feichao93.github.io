# react 16.3 以前的 context 机制实现的相关笔记

_[TODO] 是否有一个比较好的方法, 看到 React 各个组件如何处理/传递 context??_

[官方文档的介绍](https://facebook.github.io/react/docs/context.html)

> With React, it's easy to track the flow of data through your React components. When you look at a component, you can see which props are being passed, which makes your apps easy to reason about.
>
> In some cases, you want to pass data through the component tree without having to pass the props down manually at every level. You can do this directly in React with the powerful "context" API.

context 是 React 中比较高级的用法, 通过 context API, 开发者不需要进行层层传递, 就可以将数据传递给(深层的)子组件. 很多流行的类库都使用 context 来进行数据传递, 简化应用代码. 例如在 react-redux 中, [Provider](https://github.com/reactjs/react-redux/blob/a5e45d9806492d0fe1354437111722cf15b17b4d/src/components/Provider.js#L26)将 store 放在 context 中, 子组件通过[connect 函数](https://github.com/reactjs/react-redux/blob/f1367df6de7fc5aed4e2434d68038a517208f6da/src/components/connectAdvanced.js#L81)来 access context 中的 store. react-router 中也是大量使用了 context API, 例如[Router 组件](https://github.com/ReactTraining/react-router/blob/c5fb25dd725cc93d5e36c52e9bc9190f1d8418d2/packages/react-router/modules/Router.js#L23), [Route 组件](https://github.com/ReactTraining/react-router/blob/c5fb25dd725cc93d5e36c52e9bc9190f1d8418d2/packages/react-router/modules/Route.js#L29), [Switch 组件](https://github.com/ReactTraining/react-router/blob/c5fb25dd725cc93d5e36c52e9bc9190f1d8418d2/packages/react-router/modules/Switch.js#L11)等.

虽然 context 的用法较为复杂, 但在 react 内部其实相关代码不多. 下面我整理了一些与 context 相关的 React 源代码. 主要内容是初次渲染(mount)和更新(update)时 react 对 context 的处理代码. 如果你完全没明白这个文章在做什么, 那么强烈推荐先看一下[这篇文章](http://www.mattgreer.org/articles/react-internals-part-one-basic-rendering/).

## context 初始化

我们调用 ReactDOM.render(xxx)来将前端应用渲染到页面上. 初次渲染根组件的时候, context 会被初始化为空对象.

根组件渲染在某个 dom 节点, 如果该 dom 节点也是 react 渲染出来的话, 那么该 dom 节点对应的 react 组件已经有 context 了. 此时便调用已有组件的`_processChildContext`方法, 来获取根组件的 context. (_TODO, 需要验证这一点, 这一点是我目前自己猜的_)

## 初次渲染(mount)过程中, ReactCompositeComponent 对 context 的处理

在初次渲染的过程中, ReactCompositeComponent 会调用`_processContext(context)`来获取 publicContext. 这个 publicContext 才是用户自定义组件(PublicComponent)所看到的 context 对象. PublicCompoent 在 render 是所使用的`this.context`便是这个 publicContext. `processContext`会调用`_maskContext`方法, 该方法根据`Component.contextTypes`中包含的字段将 context 中的内容复制到一个新的对象中. `_maskContext`方法不会对`context`对象进行修改, 这个过程中 context 是 immutable 的.

```javascript
// src/renderers/shared/stack/reconciler/ReactCompositeComponent.js
/* other code */
var ReactCompositeComponent = {
  /* other code */
  mountComponent: function(/* other args */, context) {
    this._context = context;
    /* other code */
    var publicProps = this._currentElement.props;
    var publicContext = this._processContext(context);
    /* other code */
  }
  /* other code */
  _processContext: function(context) {
    var maskedContext = this._maskContext(context);
    if (__DEV__) {
      // 如果是DEV环境的话, 使用contextTypes对maskedContext进行验证
      /* other code */
    }
    return maskedContext;
  },
  _maskContext: function(context) {
    var Component = this._currentElement.type;
    var contextTypes = Component.contextTypes;
    if (!contextTypes) {
      return emptyObject;
    }
    var maskedContext = {};
    for (var contextName in contextTypes) {
      maskedContext[contextName] = context[contextName];
    }
    return maskedContext;
  },
  /* other code */
}
/* other code */
```

## PublicComponent 的构造函数与 context

前面讲到 ReactCompositeComponent 会计算出 publiContext, 而该 publicContext 会作为参数传入 PublicComponent 的构造函数中:

1.  如果 PublicComponent 是一个用 class 语法定义的组件, 那么使用`new Component(publicProps, publicContext, updateQueue)`来生成 PublicComponentInstance
2.  如果 PublicComponent 是 Stateless Functional Component, 那么用`Component(publicProps, publicContext, updateQueue)`来调用 Component 函数.

## ReactCompositeComponent 将 context 传递给子组件

ReactCompositeComponent 在加载子组件的时候会将`this._processChildContext(context)`作为子组件的 mountComponent 函数的 context 参数传递下去.

函数`_processChildContext`等价于 `return Object.assign({}, context, inst.getChildContext())`. 该方法返回了一个新的对象(而不是对 context 对象进行原地修改), 该对象包括了父组件 context 的内容, 也包括了组件向子组件传递的内容, 且后者会覆盖前者.

```javascript
// src/renderers/shared/stack/reconciler/ReactCompositeComponent.js
/* other code */
var ReactCompositeComponent = {
  _processChildContext: function(currentContext) {
    var Component = this._currentElement.type
    var inst = this._instance
    var childContext

    if (inst.getChildContext) {
      if (__DEV__) {
        /* other code */
      } else {
        childContext = inst.getChildContext()
      }
    }

    if (childContext) {
      /* lots of code for checking and debugging */
      return Object.assign({}, currentContext, childContext)
    }
    return currentContext
  },
}
/* other code */
```

## 更新(update)过程中, ReactCompositeComponent 对 context 的处理

更新过程中, ReactCompositeComponent 接收来父组件新的 context, 需要计算出自身的新的 publicContext. 参数 nextUnmaskedContext 对应组件的新的 context, 这里主要的操作就是判断`this._context === nextUnmaskedContext`是否成立, 如果成立的话, 直接使用复用上次的 context 即可, 否则以`nextUnmaskedContext`为参数来调用`_processContext`函数获取新的 publicContext.

```javascript
// src/renderers/shared/stack/reconciler/ReactCompositeComponent.js
/* other code */
var ReactCompositeComponent = {
  updateComponent: function(
    transaction,
    prevParentElement,
    nextParentElement,
    prevUnmaskedContext,
    nextUnmaskedContext,
  ) {
    var inst = this._instance
    invariant(/* other code */)

    var willReceive = false
    var nextContext

    // Determine if the context has changed or not
    if (this._context === nextUnmaskedContext) {
      // 用 === 就可以来判断相等性了, 因为我们不会对context进行原地修改
      // 如果context没有变化, 那么也不需要调用_processContext来处理context了
      nextContext = inst.context
    } else {
      nextContext = this._processContext(nextUnmaskedContext)
      willReceive = true
    }
    // nextContext将会变为新的context
    /* other code */
    this._context = nextUnmaskedContext
    /* other code */
  },
}
/* other code */
```

## 组件的生命周期函数与 context

首先可以在官方文档上查看[各个生命周期函数的参数约定](https://facebook.github.io/react/docs/context.html#referencing-context-in-lifecycle-methods), 每个生命周期函数都会 react 源码里面有对应的调用, 注意调用时 context 参数的传递即可.

## ReactDOMComponent 对 context 的处理

因为 ReactDOMComponent 并没有对应的 PublicComponent, 所以 ReactDOMComponent 本身并不会修改或使用 context, 那么只需要将 context 原样传递给子组件即可.
