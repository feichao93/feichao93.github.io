# react-svg-canvas 相关笔记

## 1. 渲染性能优化之 Pre-render to an off-screen canvas

对于一些不变的组件(例如坦克大战中的森林元素`<Forest />`组件, 雪地`<Snow />`组件等), 每次渲染得到的图像是完全相同的, 我们可以先将组件绘制在一个off-screen canvas, 后续需要绘制这些组件时, 调用`ctx.drawImage(offScreenCanvas, dx, dy)`来进行绘制. 参考[Pre-render to an off-screen canvas](https://www.html5rocks.com/en/tutorials/canvas/performance/#toc-pre-render)

相比正常的流程, pre-render的不同点在于:

1. 需要首先调用一次组件的render方法来获取off-screen canvas的内容, 初次调用render时的props/state需要由用户指定
2. 在获取off-screen canvas之后, 后面就不需要调用组件的render方法了, 而是改为调用`ctx.drawImage(offScreenCanvas, dx, dy)`. `dx`与`dy`的值一般取决组件的props/state, 所以我们需要让用户来指定如何调用`ctx.drawImage`

### **使用方式**

在react-svg-canvas中, 通过在组件中添加`offScreen`属性来启用off-screen pre-rendering机制, 用法如下:

```react
export default class Snow extends PureComponent {
  static offScreen = {
    // width/height指定了offScreen的尺寸
    width: 16,
    height: 16,
    
    // 为了产生offScreenCanvas, 我们会首先调用一次常规的render函数来获取组件渲染的内容
    // 这里initProps, initState将会作为首次调用render时的props/state
    initProps: { x: 0, y: 0 },
    initState: null,
    
    // 在有了offScreenCanvas之后, 后续需要绘制的时候会调用这个offScreen.render方法
    // 该render方法接受两个参数
    //   第一个ctx是我们展现内容的canvas的CanvasRenderingContext2D
    //   第二个参数offScreenCanvas是之前生成的offScreenCanvas
    //   该函数被调用的时候this指向组件本身, 所以可以通过this.props, this.state获取所需要的数据
    render(ctx, offScreenCanvas) {
      const { x, y } = this.props
      ctx.drawImage(offScreenCanvas, x, y)
    },
  }

  render() {
    const { x, y } = this.props
    return (
      <g role="snow" transform={`translate(${x},${y})`}>
        {/* snow由4个snowPart组成 */}
        {snowPart(0, 0)}
        {snowPart(8, 0)}
        {snowPart(8, 8)}
        {snowPart(0, 8)}
      </g>
    )
  }
}
```

### **具体实现**

要使用off-screen pre-rendering, 组件必须足够简单. 那些复杂的组件因为内容复杂多变, 无法使用off-screen pre-rendering.

要使用off-screen pre-rendering, **组件必须满足以下要求**: 组件的props/state发生变化时, 组件渲染的内容保持不变, 但内容的尺寸和位置可以发生变化.

**1**

首先我们知道: 通过组件本身(例如上面的`Snow`)就可以确定一个组件对应的off-screen canvas的内容和尺寸了. 我们实现了一个`constructOffScreenCanvasIfNeeded(Component)`用来按需构造off-screen canvas. 如果组件含有offScreen的配置对象, 那么就调用该函数来确保该组件的off-screen canvas已经生成. 该函数代码如下:

```typescript
// 我们只需要知道Component就足够了, 所以这里只需要一个参数
function constructOffScreenCanvasIfNeeded(Component: any) {
  // Component.offScreen为组件的off-screen canvas的配置对象, 可以参考上面Snow组件的例子
  // Component._offScreenCanvas存放了已生成的off-screen canvas
  if (Component.offScreen && Component._offScreenCanvas == null) {
    // 读取配置信息
    const { width, height, initProps, initState } = Component.offScreen

    // 生成offScreen内容与offScreenCanvas
    const offScreenCanvas = document.createElement('canvas')
    offScreenCanvas.width = width
    offScreenCanvas.height = height
    // 生成Component的实例, 并设置其props和state
    const tempInst = new Component(initProps)
    tempInst.props = initProps
    tempInst.state = initState
    // 调用组件的render方法来获取off-screen canvas的内容
    const offScreenContentElement = tempInst.render() as Element

    // 在off-screen canvas上加载offScreenContentElement, 并立刻调用internalInstance.draw()在off-screen canvas绘制内容
    const internalInstance = instantiateRscComponent(offScreenContentElement)
    RscReconciler.mountComponent(
      internalInstance,
      offScreenCanvas.getContext('2d'),
      null,
    )
    internalInstance.draw()
    // internalInstance.draw() 调用返回之后, 这里的offScreenCanvas上面已经是包含内容的了
    
    // 添加到Component._offScreenCanvas, 下次通过该字段就可以获取pre-render的内容了
    Component._offScreenCanvas = offScreenCanvas
  }
}
```



**2** 其次目前限定了比较简单的组件才有off-screen pre-rendering. 从InternalInstance的角度看, off-screen pre-rendering的内部组件的父组件肯定是RscCompositeComponent. 我们首先来看RscCompositeComponent#mountComponent代码的一部分:

```typescript
// 下面的代码是 RscCompositeComponent#mountComponent 的一部分
// 注意这一块代码是off-screen pre-rendering实现*之前*的代码
let inst: PublicComponent
if (Component.prototype && Component.prototype.isReactComponent) {
  inst = new Component(publicProps, publicContext)
} else { // stateless-functional-components
  inst = new StatelessComponent() as PublicComponent
}

inst.props = publicProps
// inst.state 已经在inst的构造函数中设置好了
inst.context = publicContext

const renderedElement = inst.render() as Element
this._renderedComponent = instantiateRscComponent(renderedElement)

RscReconciler.mountComponent(
  this._renderedComponent,
  ctx,
  this.processChildContext(context),
)
```

可以看到实现off-screen pre-rendering之前, RscCompositeComponent在加载组件的时候主要逻辑是生成Public Component实例`inst`, 然后调用`inst.render()`来获取`renderedElement`, 然后通过`instantiateRscComponent`函数生成对应的Internal Component. 但在off-screen pre-rendering机制中, 我们应该使用`offScreen.render`来进行组件的加载, 而不是总是使用`inst.render()`. 为了实现off-screen pre-rendering, 我们需要对这部分代码进行修改.



**3** 修改这部分代码其实也不麻烦, 我们首先通过`Component.offScreen`来判断该组件是否使用off-screen pre-rendering. 如果使用, 那么就调用一下`constructOffScreenCanvasIfNeeded`函数确保off-screen canvas已经准备就绪, 然后新建一个`RscOffScreenComponent`实例作为`this._renderedComponent`. 上述代码修改后如下:

```typescript
// 下面的代码是 RscCompositeComponent#mountComponent 的一部分
let inst: PublicComponent = new ...
const useOffScreenCanvas = Component.offScreen

if (useOffScreenCanvas) {
  constructOffScreenCanvasIfNeeded(Component)
  this._renderedComponent = new RscOffScreenComponent(Component)
} else {
  const renderedElement = inst.render() as Element
  this._renderedComponent = instantiateRscComponent(renderedElement)
}

RscReconciler.mountComponent(
  this._renderedComponent,
  ctx,
  this.processChildContext(context),
)
```



**4** `RscOffScreenComponent`类是我们新增的一个类. 该类的主要职责是在调用其`draw`方法时, 能将先前的off-screen canvas的内容绘制到on-screen canvas上. 同时, 该类还要实现InternalComponent的相关接口, 且不能破坏reconciliation过程. `RscOffScreenComponent`其实包含的逻辑并不多, 更多的只是将各个信息记录到实例的字段中. 代码如下:

```typescript
class RscOffScreenComponent implements InternalComponent {
  ctx: Ctx
  publicComponent: any
  // 在RscOffScreenComponent中_currentElement没用
  _currentElement: Element
  _parentComponent: InternalComponent

  // 注意这里的constructor和其他几个InternalComponent类不一样
  // 这里的参数为publicComponent, 因为publicComponent上面存放了我们所需要的数据
  // publicComponent._offScreenCanvas存放了已经绘制好的off-screen canvas
  // publicComponent.offScreen.render指向用户自定的offScreen.render方法
  constructor(publicComponent: any) {
    this.publicComponent = publicComponent
  }

  mountComponent(ctx: Ctx, context: any) {
    this.ctx = ctx
  }

  receiveComponent() { }

  unmountComponent() {
    this.ctx = null
    this.publicComponent = null
  }

  draw() { /* draw方法的代码在下面会讲到 */  }
}
```



**5** `RscOffScreenComponent#draw()`方法代码如下. 该方法获取记录下来的各个信息, 得到`PublicComponent.offScreen.render`方法调用所需的各个参数, 并调用该函数.

```typescript
function draw() {
  const ctx = this.ctx
  const offScreenRenderFn = this.publicComponent.offScreen.render
  const offScreenCanvas = this.publicComponent._offScreenCanvas
  const parentComponent = this._parentComponent as RscCompositeComponent
  offScreenRenderFn.call(parentComponent._instance, ctx, offScreenCanvas)
}
```



**6** 到目前为止, 我们实现了`constructOffScreenCanvasIfNeeded`, 修改了`RscCompositeComponent#mountComponent`方法, 新增了`RscOffScreenComponent`类, 初次渲染时的off-screen pre-rendering机制已经实现了. (如果你看过相关React文章[1][react-articles-1], [2][react-articles-2]的话, 会发现这些文章也会把react渲染流程分为*初次渲染*和*后续更新*两个部分).

接下来我们来实现*后续更新*时的off-screen pre-rendering机制. 因为进行off-screen pre-rendering的组件比较简单, 其更新的时候内容保持不变(尺寸和位置也许会发生变化), 所以对于这些组件, 我们用`RscReconciler.receiveComponent`就足够了(而不是'先卸载原来的组件, 然后再加载新的组件').

举个例子, 一个组件初次渲染得到一个矩形, 且该组件使用了off-screen pre-rendering, 那么该组件在后续更新的时候也应当返回一个矩形. 如果后续更新返回一个非矩形(例如圆形), 那么根本做不到off-screen pre-rendering. 所以对于使用off-screen pre-rendering机制的组件, 其在更新的时候, 我们总是使用`RscReconciler.receiveComponent`.

## 2. SVG与Canvas在样式上的不同点

1. SVG元素默认的fill为'black', Canvas需要调用fillxxx来绘制图形
2. SVG的fill允许使用none来表示不进行绘制, 此时Canvas则需要避免调用fillxxx
3. SVG元素默认的stroke为'none', 表示不进行描边, 此时Canvas需要避免调用strokexxx
4. SVG元素可以通过设置CSS来决定是否显示 style.visibility='hidden'   style.display='none'



[react-articles-1]: http://www.mattgreer.org/articles/react-internals-part-one-basic-rendering/	"React Internals"
[react-articels-2]: http://purplebamboo.github.io/2015/09/15/reactjs_source_analyze_part_one/	"reactjs源码分析"