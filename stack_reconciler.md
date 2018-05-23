本篇文章是官方文档的翻译，英文原文请访问[官网](https://reactjs.org/docs/implementation-notes.html)

这个章节是[stack reconciler](https://reactjs.org/docs/codebase-overview.html#stack-reconciler)的一些实现说明.

它的技术性很强并假定你能完全理解React的公开API，以及它是如何划分为核心、渲染器和协调器的。如果你对React代码不是很熟悉，请先阅读[代码概览](https://reactjs.org/docs/codebase-overview.html)。

它还假定你能够理解[React组件、实例和元素](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)的区别。


Stack reconciler 被用在React 15 以及更早的版本中， 它在源代码中的位置是[src/renderers/shared/stack/reconciler](https://github.com/facebook/react/tree/15-stable/src/renderers/shared/stack/reconciler).

### 视频：从零开始构建React

[Paul O'Shannessy](https://twitter.com/zpao)给出了一个关于[从零开始构建React](https://www.youtube.com/watch?v=_MAD4Oly9yg)的讨论，在很大程度上对本文档给予了启发。

本文档与上边的视频都是对实际代码库的简化，因此你可以通过熟悉两者来更好地理解。

### 概述

协调器本身没有公共 API. 但是诸如React DOM 和React Native的[渲染器](https://reactjs.org/docs/codebase-overview.html#stack-renderers)使用它依据用户所编写的React组件来有效地更新用户界面.

### 以递归过程的形式装载

让我们考虑首次装载组件的情形：

```js
ReactDOM.render(<App />, rootEl);
```

React DOM会将 `<App />`传递给协调器。请记住， `<App />`是一个React元素，也就是说是对哪些要渲染的东西的说明。你可以把它看成一个普通的对象：

```js
console.log(<App />);
// { type: App, props: {} }
```

协调器（reconciler）会检查 `App`是类还是函数。如果 `App` 是函数，协调器会调用`App(props)`来获取所渲染的元素。如果`App`是类，协调器则会使用`new App(props)`创建一个`App`实例，调用 `componentWillMount()` 生命周期方法，进而调用 `render()` 方法来获取所渲染的元素。无论如何，协调器都会学习App元素的“渲染行为”。

此过程是递归的。`App` 可能渲染为`<Greeting />`，而`<Greeting />`可能渲染为 `<Button />`，如此类推。因为协调器会依次学习他们各自将如何渲染，所以协调器会递归地“向下钻取”所有用户定义组件。

你可以通过如下伪代码来理解该过程：

```js
function isClass(type) {
  // React.Component的子类都会含有这一标志
  return (
    Boolean(type.prototype) &&
    Boolean(type.prototype.isReactComponent)
  );
}

// This function takes a React element (e.g. <App />)
// and returns a DOM or Native node representing the mounted tree.
// 此函数读取一个React元素（例如<App />）
// 并返回一个表达所装载树的DOM或内部节点。
function mount(element) {
  var type = element.type;
  var props = element.props;

  // 我们以此判断所渲染元素：
  // 是以函数型运行该类型
  // 还是创建新实例并调用render()。
  var renderedElement;
  if (isClass(type)) {
    // Component class
    var publicInstance = new type(props);
    // Set the props
    publicInstance.props = props;
    // Call the lifecycle if necessary
    if (publicInstance.componentWillMount) {
      publicInstance.componentWillMount();
    }
    // 调用render()以获取所渲染元素
    renderedElement = publicInstance.render();
  } else {
    // 组件函数
    renderedElement = type(props);
  }

  // 该过程是递归实现，原因在于组件可能返回一个其它组件类型的元素。
  return mount(renderedElement);

  // 注意：该实现不完整，且将无穷递归！ 它只处理<App />或<Button />等元素。尚不处理<div />或<p />等元素。
}

var rootEl = document.getElementById('root');
var node = mount(<App />);
rootEl.appendChild(node);
```

>**注意:**
>
>这的确是一段伪代码。它与真实的实现不同。它会导致栈溢出，因为我们还没有讨论何时停止递归。

让我们回顾一下上面示例中的几个关键概念:

* React元素是表示组件类型（例如 `App`）与属性的普通对象。
* 用户定义组件（例如 `App`）可以为类或者函数，但它们都会被“渲染为”元素。
* “装载”（Mounting）是一个递归过程，当给定顶级React元素（例如<App />）时创建DOM或内部节点树。

### 装载主机元素（Mounting Host Elements）

该过程将没有任何意义，如果最终没有渲染内容到屏幕上。

除了用户定义的（“复合”）组件外, React元素还可能表示特定于平台的（“主机”）组件。例如，`Button`可能会从其渲染方法中返回 `<div />` 。

如果元素的type属性是一个字符串，即表示我们正在处理一个主机元素（host element）：

```js
console.log(<div />);
// { type: 'div', props: {} }
```

主机元素（host elements）不存在关联的用户定义代码。

当协调器遇到主机元素（host element）时，它会让渲染器（renderer）装载它（mounting）。例如，React DOM将会创建一个DOM节点。

如果主机元素（host element）有子级，协调器（reconciler）则会用上述相同算法递归地将它们装载。而不管子级是主机元素（如`<div><hr /></div>`）还是混合元素（如`<div><Button /></div>`）或是两者兼有。

由子级组件生成的DOM节点将被追加到DOM父节点，同时整的DOM结构会被递归装配。

>**注意:**
>
>协调器本身(reconciler)并不与DOM捆绑。装载(mounting)的具体结果（有时在源代码中称为“装载映像”）取决于渲染器(renderer)，可能为 DOM节点（React DOM）、字符串（React DOM服务器）或表示本机视图的数值（React Native）。


我们来扩展一下代码，以处理主机元素（host elements）:

```js
function isClass(type) {
  //  React.Component 子类含有这一标志
  return (
    Boolean(type.prototype) &&
    Boolean(type.prototype.isReactComponent)
  );
}

// 该函数仅处理含复合类型的元素。  例如，它处理<App />和<Button />，但不处理<div />。
function mountComposite(element) {
  var type = element.type;
  var props = element.props;

  var renderedElement;
  if (isClass(type)) {
    // 组件类
    var publicInstance = new type(props);
    // 设置属性
    publicInstance.props = props;
    // 若必要，则调用生命周期函数
    if (publicInstance.componentWillMount) {
      publicInstance.componentWillMount();
    }
    renderedElement = publicInstance.render();
  } else if (typeof type === 'function') {
    // 组件函数
    renderedElement = type(props);
  }

  // 该过程是递归，一旦该元素为主机（如<div />｝而非复合（如<App />）时，则逐渐结束
  return mount(renderedElement);
}

// 该函数仅处理含主机类型的元素(handles elements with a host type)。 例如，它处理<div />和<p />但不处理<App />。
function mountHost(element) {
  var type = element.type;
  var props = element.props;
  var children = props.children || [];
  if (!Array.isArray(children)) {
    children = [children];
  }
  children = children.filter(Boolean);

  // 该代码块不可出现在协调器(reconciler)中。
  // 不同渲染器(renderers)可能会以不同方式初始化节点。
  // 例如，React Native会生成iOS或Android视图。
  var node = document.createElement(type);
  Object.keys(props).forEach(propName => {
    if (propName !== 'children') {
      node.setAttribute(propName, props[propName]);
    }
  });

  // 装载子节点
  children.forEach(childElement => {
    // 子节点有可能是主机元素（如<div />）或复合元素（如<Button />）.
    // 所以我们应该递归的装载
    var childNode = mount(childElement);

    // 此行代码仍是特定于渲染器的。不同的渲染器则会使用不同的方法
    node.appendChild(childNode);
  });

  // 返回DOM节点作为装载结果
  // 此处即为递归结束.
  return node;
}

function mount(element) {
  var type = element.type;
  if (typeof type === 'function') {
    // 用户定义的组件
    return mountComposite(element);
  } else if (typeof type === 'string') {
    // 平台相关的组件，比如说浏览器中的div，ios和安卓中的视图
    return mountHost(element);
  }
}

var rootEl = document.getElementById('root');
var node = mount(<App />);
rootEl.appendChild(node);
```

该代码能够工作但仍与协调器（reconciler）的真正实现相差甚远。其所缺少的关键部分是对更新的支持。

### 介绍内部实例

React 的关键特征是您可以重新渲染所有内容, 它不会重新创建 DOM 或重置状态:

```js
ReactDOM.render(<App />, rootEl);
// 应该重新使用现存的 DOM:
ReactDOM.render(<App />, rootEl);
```

但是, 上面的实现只知道如何装载初始树。它无法对其执行更新, 因为它没有存储所有必需的信息, 例如所有 `publicInstance` , 
或者哪个 DOM 节点 对应于哪些组件。

堆栈协调（stack reconciler）的基本代码是通过使 mount () 函数成为一个方法并将其放在类上来解决这一问题。
这种方式有一些缺陷，但是目前代码中仍然使用的是这种方式。不过目前我们也正在重写[协调器（reconciler）](https://reactjs.org/docs/codebase-overview.html#fiber-reconciler)

我们将创建两个类: DOMComponent 和 CompositeComponent , 而不是单独的 mountHost 和 mountComposite 函数。

两个类都有一个接受 element 的构造函数, 以及一个能返回已装入节点的 mount () 方法。我们将用一个能实例化正确类的工厂函数替换掉之前
例子里的mount函数：

```js
function instantiateComponent(element) {
  var type = element.type;
  if (typeof type === 'function') {
    // 用户自定义组件
    return new CompositeComponent(element);
  } else if (typeof type === 'string') {
    // 特定于平台的组件
    return new DOMComponent(element);
  }  
}
```

首先, 让我们考虑如何实现 `CompositeComponent`:

```js
class CompositeComponent {
  constructor(element) {
    this.currentElement = element;
    this.renderedComponent = null;
    this.publicInstance = null;
  }

  getPublicInstance() {
    // 针对复合组合, 返回类的实例.
    return this.publicInstance;
  }

  mount() {
    var element = this.currentElement;
    var type = element.type;
    var props = element.props;

    var publicInstance;
    var renderedElement;
    if (isClass(type)) {
      // 组件类
      publicInstance = new type(props);
      // 设置属性
      publicInstance.props = props;
      // 如果有必要，调用生命周期
      if (publicInstance.componentWillMount) {
        publicInstance.componentWillMount();
      }
      renderedElement = publicInstance.render();
    } else if (typeof type === 'function') {
      // Component function
      publicInstance = null;
      renderedElement = type(props);
    }

    // Save the public instance
    this.publicInstance = publicInstance;

    // 通过element实例化内部的child实例，这个实例有可能是DOMComponent，比如<div /> or <p />
    // 也可能是CompositeComponent 比如说<App /> or <Button />
    var renderedComponent = instantiateComponent(renderedElement);
    this.renderedComponent = renderedComponent;

    // 增加渲染输出
    return renderedComponent.mount();
  }
}
```

这与我们以前的 `mountComposite()` 实现没有太大的不同, 但现在我们可以保存一些信息, 
比如`this.currentElement`、`this.renderedComponent` 和 `this.publicInstance` ,这些保存的信息会在更新期间被使用。

请注意, `CompositeComponent`的实例与用户提供的 `element.type` 的实例不是一回事。
`CompositeComponent`是我们的协调器（reconciler）的一个实现细节, 从不向用户公开。
用户自定义类是我们从 `element.type` 读取的，并且通过 `CompositeComponent` 创建它的一个实例。

为避免混乱，我们将`CompositeComponent`和`DOMComponent`的实例称为“内部实例”。
由于它们的存在, 我们可以将一些长寿数据（ong-lived）与它们关联起来。只有渲染器（renderer）和协调器（reconciler）知道它们的存在。

另一方面, 我们将用户定义的类的实例称为 "公共实例"（public instance）。公共实例是您在 `render()` 和自定义组件的其他方法中看到的 `this`

`mountHost()` 函数被重构为 `DOMComponent` 类上的 `mount()`方法, 也看起来很熟悉:

```js
class DOMComponent {
  constructor(element) {
    this.currentElement = element;
    this.renderedChildren = [];
    this.node = null;
  }

  getPublicInstance() {
    // For DOM components, only expose the DOM node.
    return this.node;
  }

  mount() {
    var element = this.currentElement;
    var type = element.type;
    var props = element.props;
    var children = props.children || [];
    if (!Array.isArray(children)) {
      children = [children];
    }

    // Create and save the node
    var node = document.createElement(type);
    this.node = node;

    // Set the attributes
    Object.keys(props).forEach(propName => {
      if (propName !== 'children') {
        node.setAttribute(propName, props[propName]);
      }
    });

    // Create and save the contained children.
    // Each of them can be a DOMComponent or a CompositeComponent,
    // depending on whether the element type is a string or a function.
    var renderedChildren = children.map(instantiateComponent);
    this.renderedChildren = renderedChildren;

    // Collect DOM nodes they return on mount
    var childNodes = renderedChildren.map(child => child.mount());
    childNodes.forEach(childNode => node.appendChild(childNode));

    // Return the DOM node as mount result
    return node;
  }
}
```
从 mountHost () 重构后的主要区别在于, 我们现在将 `this.node`  和 `this.renderedChildren`  与内部 DOM 组件实例相关联。
我们还将使用它们在将来应用非破坏性更新。

因此, 每个内部实例 (复合实例或主机实例)（composite or host） 现在都指向内部的子实例。为帮助可视化, 如果功能 `<App>` 组件呈现 `<Button>`  类组件, 并且  `<Button>` 类呈现`<div>`, 则内部实例树将如下所显示:

```js
[object CompositeComponent] {
  currentElement: <App />,
  publicInstance: null,
  renderedComponent: [object CompositeComponent] {
    currentElement: <Button />,
    publicInstance: [object Button],
    renderedComponent: [object DOMComponent] {
      currentElement: <div />,
      node: [object HTMLDivElement],
      renderedChildren: []
    }
  }
}
```

在 DOM 中, 您只会看到`<div>` 。但是, 内部实例树同时包含复合和主机内部实例（composite and host internal instances）。

内部的复合实例需要存储下面的信息:

* 当前元素（The current element）.
* 如果元素类型是类, 则将类实例化并存为公共实例（The public instance if element type is a class）.
* 一个通过运行render()之后并传入工厂函数而得到的内部实例（renderedComponent）。它可以是一个`DOMComponent`或一个`CompositeComponent`。

内部的主机实例需要存储下面的信息:

* 当前元素（The current element）.
* DOM 节点（The DOM node）.
* 所有的内部子实例，他们可以是 `DOMComponent` or a `CompositeComponent`。（All the child internal instances. Each of them can be either a `DOMComponent` or a `CompositeComponent`）.

如果你很难想象一个内部的实例树是如何在更复杂的应用中构建的， [React DevTools]((https://github.com/facebook/react-devtools))可以给出一个非常接近的近似，因为它突出显示了带有灰色的主机实例，以及用紫色表示的组合实例:

 <img src="../images/implementation-notes-tree.png" width="500" style="max-width: 100%" alt="React DevTools tree" />

为了完成这个重构，我们将引入一个函数，它将一个完整的树挂载到一个容器节点，就像`ReactDOM.render()`。它返回一个公共实例，也类似于 `ReactDOM.render()`:

```js
function mountTree(element, containerNode) {
  // 创建顶级内部实例
  var rootComponent = instantiateComponent(element);

  // 将顶级组件装载到容器中
  var node = rootComponent.mount();
  containerNode.appendChild(node);

  // 返回它所提供的公共实例
  var publicInstance = rootComponent.getPublicInstance();
  return publicInstance;
}

var rootEl = document.getElementById('root');
mountTree(<App />, rootEl);
```

### 卸载(Unmounting)

现在，我们有了保存有它们的子节点和DOM节点的内部实例，我们可以实现卸载。对于一个复合组件（composite component），卸载将调用一个生命周期钩子然后递归进行。

```js
class CompositeComponent {

  // ...

  unmount() {
    // Call the lifecycle hook if necessary
    var publicInstance = this.publicInstance;
    if (publicInstance) {
      if (publicInstance.componentWillUnmount) {
        publicInstance.componentWillUnmount();
      }
    }

    // Unmount the single rendered component
    var renderedComponent = this.renderedComponent;
    renderedComponent.unmount();
  }
}
```

对于`DOMComponent`，卸载操作让每个孩子进行卸载：

```js
class DOMComponent {

  // ...

  unmount() {
    // Unmount all the children
    var renderedChildren = this.renderedChildren;
    renderedChildren.forEach(child => child.unmount());
  }
}
```

在实践中，卸载DOM组件也会删除事件侦听器并清除一些缓存，为了便于理解，我们暂时跳过这些细节。

现在我们可以添加一个顶级函数，叫作`unmountTree(containerNode)`，它与`ReactDOM.unmountComponentAtNode()`类似:

```js
function unmountTree(containerNode) {
  // Read the internal instance from a DOM node:
  // (This doesn't work yet, we will need to change mountTree() to store it.)
  var node = containerNode.firstChild;
  var rootComponent = node._internalInstance;

  // Unmount the tree and clear the container
  rootComponent.unmount();
  containerNode.innerHTML = '';
}
```

为了使其工作，我们需要从一个DOM节点读取一个内部根实例。我们将修改 `mountTree()`  以将 `_internalInstance` 属性添加到DOM 根节点。
我们也将教`mountTree()`去销毁任何现存树，以便将来它可以被多次调用：

```js
function mountTree(element, containerNode) {
  // Destroy any existing tree
  if (containerNode.firstChild) {
    unmountTree(containerNode);
  }

  // Create the top-level internal instance
  var rootComponent = instantiateComponent(element);

  // Mount the top-level component into the container
  var node = rootComponent.mount();
  containerNode.appendChild(node);

  // Save a reference to the internal instance
  node._internalInstance = rootComponent;

  // Return the public instance it provides
  var publicInstance = rootComponent.getPublicInstance();
  return publicInstance;
}
```

现在，可以反复运行`unmountTree()`或者 `mountTree()`，清除旧树并且在组件上运行 `componentWillUnmount()` 生命周期钩子。

### 更新（Updating）

在上一节中，我们实现了卸载。然而，如果每个组件的prop的变动都要卸载并挂载整个树，这是不可接受的。幸好我们设计了协调器。
协调器（reconciler）的目标是重用已存在的实例，以便保留DOM和状态:

```js
var rootEl = document.getElementById('root');

mountTree(<App />, rootEl);
// 应该重用现有的DOM:
mountTree(<App />, rootEl);
```

我们将用一种方法扩展我们的内部实例。
除了 `mount()`和 `unmount()`。`DOMComponent`和 `CompositeComponent`将实现一个新的方法，它叫作 `receive(nextElement)`:

```js
class CompositeComponent {
  // ...

  receive(nextElement) {
    // ...
  }
}

class DOMComponent {
  // ...

  receive(nextElement) {
    // ...
  }
}
```

它的工作是做任何必要的工作，以使组件(及其任何子节点) 能够根据 `nextElement` 提供的信息保持信息为最新状态。

这是经常被描述为"virtual DOM diffing"的部分，尽管真正发生的是我们递归地遍历内部树，并让每个内部实例接收到更新指令。

### 更新复合组件（Updating Composite Components）

当一个复合组件接收到一个新元素（element）时，我们运行componentWillUpdate()生命周期钩子。

然后，我们使用新的props重新render组件，并获得下一个render的元素（rendered element）：

```js
class CompositeComponent {

  // ...

  receive(nextElement) {
    var prevProps = this.currentElement.props;
    var publicInstance = this.publicInstance;
    var prevRenderedComponent = this.renderedComponent;
    var prevRenderedElement = prevRenderedComponent.currentElement;

    // Update *own* element
    this.currentElement = nextElement;
    var type = nextElement.type;
    var nextProps = nextElement.props;

    // Figure out what the next render() output is
    var nextRenderedElement;
    if (isClass(type)) {
      // Component class
      // Call the lifecycle if necessary
      if (publicInstance.componentWillUpdate) {
        publicInstance.componentWillUpdate(nextProps);
      }
      // Update the props
      publicInstance.props = nextProps;
      // Re-render
      nextRenderedElement = publicInstance.render();
    } else if (typeof type === 'function') {
      // Component function
      nextRenderedElement = type(nextProps);
    }

    // ...
```

下一步，我们可以看一下渲染元素的type。如果自从上次渲染，type 没有被改变，组件接下来可以被适当更新。

例如，如果它第一次返回 `<Button color="red" />`，并且第二次返回 `<Button color="blue" />`，我们可以告诉内部实例去 `receive()` 下一个元素：

```js
    // ...

    // 如果被渲染元素类型没有被改变,
    // 重用现有的组件实例.
    if (prevRenderedElement.type === nextRenderedElement.type) {
      prevRenderedComponent.receive(nextRenderedElement);
      return;
    }

    // ...

```

但是，如果下一个被渲染元素和前一个相比有一个不同的`type` ，我们不能更新内部实例。因为一个 <button> 不“能变”为一个 <input>.

相反，我们必须卸载现有的内部实例并挂载对应于渲染的元素类型的新实例。
例如，这就是当一个之前被渲染的元素`<button />`之后又被渲染成一个  `<input />` 的过程:

```js
    // ...

    // If we reached this point, we need to unmount the previously
    // mounted component, mount the new one, and swap their nodes.

    // Find the old node because it will need to be replaced
    var prevNode = prevRenderedComponent.getHostNode();

    // Unmount the old child and mount a new child
    prevRenderedComponent.unmount();
    var nextRenderedComponent = instantiateComponent(nextRenderedElement);
    var nextNode = nextRenderedComponent.mount();

    // Replace the reference to the child
    this.renderedComponent = nextRenderedComponent;

    // Replace the old node with the new one
    // Note: this is renderer-specific code and
    // ideally should live outside of CompositeComponent:
    prevNode.parentNode.replaceChild(nextNode, prevNode);
  }
}
```
总而言之，当一个复合组件（composite component）接收到一个新元素时，它可能会将更新委托给其渲染的内部实例（（rendered internal instance），
或者卸载它，并在其位置上挂一个新元素。

另一种情况下，组件将重新挂载而不是接收一个元素，并且这发生在元素的`key`变化时。本文档中，我们不讨论key 处理，因为它将使原本复杂的教程更加复杂。

注意，我们需要添加一个叫作`getHostNode()`的新方法到内部实例（internal instance），以便可以定位特定于平台的节点并在更新期间替换它。
它的实现对两个类都很简单：

```js
class CompositeComponent {
  // ...

  getHostNode() {
    // 请求渲染的组件提供它（Ask the rendered component to provide it）.
    // 这将递归地向下钻取任何组合(This will recursively drill down any composites).
    return this.renderedComponent.getHostNode();
  }
}

class DOMComponent {
  // ...

  getHostNode() {
    return this.node;
  }  
}
```

### 更新主机组件(Updating Host Components)

主机组件实现(例如DOMComponent), 是以不同方式更新.当它们接收到一个元素时，它们需要更新底层特定于平台的视图。在 React DOM 中，这意味着更新 DOM 属性：

```js
class DOMComponent {
  // ...

  receive(nextElement) {
    var node = this.node;
    var prevElement = this.currentElement;
    var prevProps = prevElement.props;
    var nextProps = nextElement.props;    
    this.currentElement = nextElement;

    // Remove old attributes.
    Object.keys(prevProps).forEach(propName => {
      if (propName !== 'children' && !nextProps.hasOwnProperty(propName)) {
        node.removeAttribute(propName);
      }
    });
    // Set next attributes.
    Object.keys(nextProps).forEach(propName => {
      if (propName !== 'children') {
        node.setAttribute(propName, nextProps[propName]);
      }
    });

    // ...
```

接下来，主机组件需要更新它们的子元素。与复合组件不同的是，它们可能包含多个子元素。

在这个简化的例子中，我们使用一个内部实例的数组并对其进行迭代，是更新或替换内部实例，这取决于接收到的type是否与之前的type匹配。
真正的调解器（reconciler）同时在帐户中获取元素的key并且追踪变动，除了插入与删除，但是我们现在先忽略这一逻辑。

我们在列表中收集DOM操作，这样我们就可以批量地执行它们。

```js
    // ...

    // // 这些是React元素（element）数组:
    var prevChildren = prevProps.children || [];
    if (!Array.isArray(prevChildren)) {
      prevChildren = [prevChildren];
    }
    var nextChildren = nextProps.children || [];
    if (!Array.isArray(nextChildren)) {
      nextChildren = [nextChildren];
    }
    // 这些是内部实例(internal instances)数组:
    var prevRenderedChildren = this.renderedChildren;
    var nextRenderedChildren = [];

    // 当我们遍历children时，我们将向数组中添加操作。
    var operationQueue = [];

    // 注意：以下章节大大减化！
    // 它不处理reorders，空children，或者keys。
    // 它只是用来解释整个流程，而不是具体的细节。

    for (var i = 0; i < nextChildren.length; i++) {
      // 尝试为这个子级获取现存内部实例。
      var prevChild = prevRenderedChildren[i];

      // 如果在这个索引下没有内部实例,那说明是一个child被添加了末尾。
      // 这时应该去创建一个内部实例，挂载它，并使用它的节点。
      if (!prevChild) {
        var nextChild = instantiateComponent(nextChildren[i]);
        var node = nextChild.mount();

        // 记录一下我们将来需要append一个节点（node）
        operationQueue.push({type: 'ADD', node});
        nextRenderedChildren.push(nextChild);
        continue;
      }

      // 如果它的元素类型匹配，我们只需要更新该实例即可  
      // 例如, <Button size="small" /> 可以更新为
      // <Button size="large" /> 但是不能被更新为 <App />.
      var canUpdate = prevChildren[i].type === nextChildren[i].type;

      // 如果我们不能更新现有的实例，我们就必须卸载它。然后装一个新的替代它。
      if (!canUpdate) {
        var prevNode = prevChild.getHostNode();
        prevChild.unmount();

        var nextChild = instantiateComponent(nextChildren[i]);
        var nextNode = nextChild.mount();

        // 记录一下我们将来需要替换这些nodes
        operationQueue.push({type: 'REPLACE', prevNode, nextNode});
        nextRenderedChildren.push(nextChild);
        continue;
      }

      // 如果我们可以更新现存的内部实例（internal instance），
      // 我们仅仅把下一个元素传入其receive即可，让其receive函数处理它的更新即可
      prevChild.receive(nextChildren[i]);
      nextRenderedChildren.push(prevChild);
    }

    // 最后，卸载（unmount）哪些不存在的children
    for (var j = nextChildren.length; j < prevChildren.length; j++) {
      var prevChild = prevRenderedChildren[j];
      var node = prevChild.getHostNode();
      prevChild.unmount();

      // 记录一下我们将来需要remove这些node
      operationQueue.push({type: 'REMOVE', node});
    }

    // Point the list of rendered children to the updated version.
    this.renderedChildren = nextRenderedChildren;

    // ...
```

作为最后一步，我们执行DOM操作。还是那句话，真正的协调器（reconciler）代码更复杂，因为它还能处理移动：

```js
    // ...

    // 处理队列里的operation。
    while (operationQueue.length > 0) {
      var operation = operationQueue.shift();
      switch (operation.type) {
      case 'ADD':
        this.node.appendChild(operation.node);
        break;
      case 'REPLACE':
        this.node.replaceChild(operation.nextNode, operation.prevNode);
        break;
      case 'REMOVE':
        this.node.removeChild(operation.node);
        break;
      }
    }
  }
}
```

这是用来更新主机组件（host components）的。

### 顶级更新（Top-Level Updates）

现在 CompositeComponent 与 DOMComponent 都实现了 receive(nextElement) 方法, 
我们现在可以改变顶级 mountTree() 函数了，当元素（element）的`type`相同时，我们可以使用receive了。

```js
function mountTree(element, containerNode) {
  // Check for an existing tree
  if (containerNode.firstChild) {
    var prevNode = containerNode.firstChild;
    var prevRootComponent = prevNode._internalInstance;
    var prevElement = prevRootComponent.currentElement;

    // 如果可以，使用现存根组件
    if (prevElement.type === element.type) {
      prevRootComponent.receive(element);
      return;
    }

    // 否则，卸载现存树
    unmountTree(containerNode);
  }

  // ...

}
```

现在调用 mountTree()两次，同样的类型不会先卸载再装载了：

```js
var rootEl = document.getElementById('root');

mountTree(<App />, rootEl);
// 复用现存 DOM:
mountTree(<App />, rootEl);
```

These are the basics of how React works internally.

### 我们遗漏的还有什么？

与真正的代码库相比，这个文档被简化了。有一些重要的方面我们没有提到：

* 组件可以渲染null，而且，协调器（reconciler）可以处理数组中的“空槽（empty slots）”并显示输出。

* 协调器（reconciler）可以从元素中读取 key ，并且用它来建立在一个数组中内部实例与元素的对应关系。实际的 React 实现的大部分复杂性与此相关。

* 除了复合和主机内部实例类之外，还存在用于“文本”和“空”组件的类。它们表示文本节点和通过渲染 null得到的“空槽”。

* 渲染器（Renderers）使用[injection](https://reactjs.org/docs/codebase-overview.html#dynamic-injection)
将主机内部类传递给协调器(reconciler)。例如，React DOM 告诉协调器使用 `ReactDOMComponent` 作为主机内部实现实例。

* 更新子列表的逻辑被提取到一个名为 `ReactMultiChild` 的mixin中，它被主机内部实例类实现在 React DOM和 React Native时都使用。

* 协调器也实现了在复合组件（composite components）中支持setState()。事件处理程序内部的多个更新将被打包成一个单一的更新。

* 协调器（reconciler）还负责复合组件和主机节点的refs。

* 在DOM准备好之后调用的生命周期钩子，例如 componentDidMount() 和 componentDidUpdate()，收集到“回调队列”，并在单个批处理中执行。

* React 将当前更新的信息放入一个名为“事务”的内部对象中。事务对于跟踪挂起的生命周期钩子的队列、
为了warning而嵌套的当前DOM（the current DOM nesting for the warnings）以及任何“全局”到特定的更新都是有用的。
事务还可以确保在更新后“清除所有内容”。例如，由 React DOM提供的事务类在任何更新之后恢复input的选中与否。

### 直接查看代码（Jumping into the Code）

* 在 [`ReactMount`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/dom/client/ReactMount.js) 中可以查看此教程中类似 `mountTree()` 和 `unmountTree()` 的代码.
它负责装载(mounting)和卸载（unmounting）顶级组件。
[`ReactNativeMount`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/native/ReactNativeMount.js) is its React Native analog.

* [`ReactDOMComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/dom/shared/ReactDOMComponent.js) 
在教程中与DOMComponent等同. 它实现了 React DOM渲染器（renderer）的主机组件类（host component class。
[`ReactNativeBaseComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/native/ReactNativeBaseComponent.js) is its React Native analog.

* [`ReactCompositeComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js) 
在教程中与 `CompositeComponent` 等同. 它处理调用用户定义的组件并维护它们的状态。

* [`instantiateReactComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/instantiateReactComponent.js)
 包含选择正确的内部实例类并运行element的构造函数。在本教程中，它与`instantiateComponent()`等同。

* [`ReactReconciler`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactReconciler.js) 
是一个具有 `mountComponent()`, `receiveComponent()`, 和 `unmountComponent()` 方法的封装.
它调用内部实例的底层实现，但也包含了所有内部实例实现共享的代码。

* [`ReactChildReconciler`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactChildReconciler.js) 
根据元素的 `key` ，实现了mounting、updating和unmounting的逻辑.

* [`ReactMultiChild`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactMultiChild.js) 
独立于渲染器的操作队列，实现了处理child的插入、删除和移动

* 由于遗留的原因 `mount()`, `receive()`, and `unmount()` 被称作 `mountComponent()`, 
`receiveComponent()`, and `unmountComponent()` 但是他们却接收elements

* 内部实例的属性以一个下划线开始, 例如， `_currentElement`. 在整个代码库中，它们被认为是只读的公共字段。

### 未来方向（Future Directions）

堆栈协调器具有固有的局限性, 如同步和无法中断工作或分割成区块。
我们正在实现一个新的协调器[Fiber reconciler](https://reactjs.org/docs/codebase-overview.html#fiber-reconciler), 
你可以在这里看它的[具体思路](https://github.com/acdlite/react-fiber-architecture)
将来我们会用fiber协调器代替stack协调器（译者注：其实现在react16已经发布，在react16中fiber算法已经取代了stack算法）

### 下一步（Next Steps）

阅读[next section](https://reactjs.org/docs/design-principles.html)以了解有关协调器的当前实现的详细信息。