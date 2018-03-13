# react 16.3以前的 context 机制实现的相关笔记

*[TODO] 是否有一个比较好的方法, 看到React各个组件如何处理/传递context??*

[官方文档的介绍](https://facebook.github.io/react/docs/context.html)

> With React, it's easy to track the flow of data through your React components. When you look at a component, you can see which props are being passed, which makes your apps easy to reason about.
>
> In some cases, you want to pass data through the component tree without having to pass the props down manually at every level. You can do this directly in React with the powerful "context" API.

context是React中比较高级的用法, 通过context API, 开发者不需要进行层层传递, 就可以将数据传递给(深层的)子组件. 很多流行的类库都使用context来进行数据传递, 简化应用代码. 例如在react-redux中, [Provider](https://github.com/reactjs/react-redux/blob/a5e45d9806492d0fe1354437111722cf15b17b4d/src/components/Provider.js#L26)将store放在context中, 子组件通过[connect函数](https://github.com/reactjs/react-redux/blob/f1367df6de7fc5aed4e2434d68038a517208f6da/src/components/connectAdvanced.js#L81)来access context中的store. react-router中也是大量使用了context API, 例如[Router组件](https://github.com/ReactTraining/react-router/blob/c5fb25dd725cc93d5e36c52e9bc9190f1d8418d2/packages/react-router/modules/Router.js#L23), [Route组件](https://github.com/ReactTraining/react-router/blob/c5fb25dd725cc93d5e36c52e9bc9190f1d8418d2/packages/react-router/modules/Route.js#L29), [Switch组件](https://github.com/ReactTraining/react-router/blob/c5fb25dd725cc93d5e36c52e9bc9190f1d8418d2/packages/react-router/modules/Switch.js#L11)等.

虽然context的用法较为复杂, 但在react内部其实相关代码不多. 下面我整理了一些与context相关的React源代码. 主要内容是初次渲染(mount)和更新(update)时react对context的处理代码. 如果你完全没明白这个文章在做什么, 那么强烈推荐先看一下[这篇文章](http://www.mattgreer.org/articles/react-internals-part-one-basic-rendering/).

## context初始化

我们调用ReactDOM.render(xxx)来将前端应用渲染到页面上. 初次渲染根组件的时候, context会被初始化为空对象.

根组件渲染在某个dom节点, 如果该dom节点也是react渲染出来的话, 那么该dom节点对应的react组件已经有context了. 此时便调用已有组件的`_processChildContext`方法, 来获取根组件的context. (*TODO, 需要验证这一点, 这一点是我目前自己猜的*)

## 初次渲染(mount)过程中, ReactCompositeComponent对context的处理

在初次渲染的过程中, ReactCompositeComponent会调用`_processContext(context)`来获取publicContext. 这个publicContext才是用户自定义组件(PublicComponent)所看到的context对象. PublicCompoent在render是所使用的`this.context`便是这个publicContext. `processContext`会调用`_maskContext`方法, 该方法根据`Component.contextTypes`中包含的字段将context中的内容复制到一个新的对象中. `_maskContext`方法不会对`context`对象进行修改, 这个过程中context是immutable的.

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
## PublicComponent的构造函数与context

前面讲到ReactCompositeComponent会计算出publiContext, 而该publicContext会作为参数传入PublicComponent的构造函数中:

1. 如果PublicComponent是一个用class语法定义的组件, 那么使用`new Component(publicProps, publicContext, updateQueue)`来生成PublicComponentInstance
2. 如果PublicComponent是Stateless Functional Component, 那么用`Component(publicProps, publicContext, updateQueue)`来调用Component函数.

## ReactCompositeComponent将context传递给子组件

ReactCompositeComponent在加载子组件的时候会将`this._processChildContext(context)`作为子组件的mountComponent函数的context参数传递下去.

函数`_processChildContext`等价于 `return Object.assign({}, context, inst.getChildContext())`. 该方法返回了一个新的对象(而不是对context对象进行原地修改), 该对象包括了父组件context的内容, 也包括了组件向子组件传递的内容, 且后者会覆盖前者.

```javascript
// src/renderers/shared/stack/reconciler/ReactCompositeComponent.js
/* other code */
var ReactCompositeComponent = {
  _processChildContext: function(currentContext) {
    var Component = this._currentElement.type;
    var inst = this._instance;
    var childContext;

    if (inst.getChildContext) {
      if (__DEV__) { /* other code */ }
      else {
        childContext = inst.getChildContext();
      }
    }

    if (childContext) {
      /* lots of code for checking and debugging */
      return Object.assign({}, currentContext, childContext);
    }
    return currentContext;
  },
}
/* other code */
```



## 更新(update)过程中, ReactCompositeComponent对context的处理 

更新过程中, ReactCompositeComponent接收来父组件新的context, 需要计算出自身的新的publicContext. 参数nextUnmaskedContext对应组件的新的context, 这里主要的操作就是判断`this._context === nextUnmaskedContext`是否成立, 如果成立的话, 直接使用复用上次的context即可, 否则以`nextUnmaskedContext`为参数来调用`_processContext`函数获取新的publicContext.

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
    var inst = this._instance;
    invariant( /* other code */ );

    var willReceive = false;
    var nextContext;

    // Determine if the context has changed or not
    if (this._context === nextUnmaskedContext) {
      // 用 === 就可以来判断相等性了, 因为我们不会对context进行原地修改
      // 如果context没有变化, 那么也不需要调用_processContext来处理context了
      nextContext = inst.context;
    } else {
      nextContext = this._processContext(nextUnmaskedContext);
      willReceive = true;
    }
    // nextContext将会变为新的context
    /* other code */
    this._context = nextUnmaskedContext;
    /* other code */
  },
}
/* other code */
```

## 组件的生命周期函数与context

首先可以在官方文档上查看[各个生命周期函数的参数约定](https://facebook.github.io/react/docs/context.html#referencing-context-in-lifecycle-methods), 每个生命周期函数都会react源码里面有对应的调用, 注意调用时context参数的传递即可.

## ReactDOMComponent对context的处理

因为ReactDOMComponent并没有对应的PublicComponent, 所以ReactDOMComponent本身并不会修改或使用context, 那么只需要将context原样传递给子组件即可.