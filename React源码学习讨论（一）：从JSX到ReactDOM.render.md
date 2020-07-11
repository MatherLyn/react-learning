## 导语
React源码一直是许多初学者想看却觉得有门槛的内容。对于初学者来说，熟悉React的特性，了解React的设计是非常好的知识提升。下面的文章将会从一个简单的React组件开始，从初学者的角度来逐步深入React源码。

读懂以下的文章需要：
- 一些JavaScript（maybe TypeScript）基础
- 曾经编写过React代码
- 一些计算机基础

接下来，就让我们一步步探索React源码。

## 一个简单的React组件

在React中，我们使用JSX/TSX进行开发。目前最流行，也是逻辑感最强的写法是用class来描述一个组件：

``` tsx
import React from 'react';

interface IProps {
  // some properties defined on props...
}

interface IState {
  msg: string;
  // some other properties defined on state...
}

class MyComponent extends React.Component<IProps, IState> {
  constructor(props: IProps) {
    super(props);
    this.state = {
      msg: 'Hello world.'
    };
  }
  
  private handleClick = () => {
    this.setState({
      msg: 'Hello React.'
    });
  }
  
  render() {
    return <div onClick={this.handleClick}>{ this.state.msg }</div>
  }
}
```

如上，便是一个最简单的React组件。听过有人说，“我的工作就是 `flex + onClick + setState` 一把梭”。其实这就像这个简单组件一样，包含了对数据的处理、显示与更新方法，这也是前端工作人员最主要的工作之一。但是刚开始上手React时，写完这个代码后会发现，其中有许多疑惑点我们是没明白的。

## 从目标代码来看这个组件做了什么

先梳理一下我们在源代码中写了什么逻辑：

定义这个类的时候，我们继承了React中的`Component`类，使用泛型为`props`和`state`定义了类型。在组件的`state`中定义了需要显示的`msg`变量，并为构造出的DOM挂载点击事件。

熟悉JavaScript / TypeScript的同学会知道，这门语言里的类本质上是一个构造函数。babel会将它进行对应的语法转义，使其返回一个所谓的JSX对象。先抛开事件`handleClick`的挂载（这个是独立的React事件模块，后面会单独挑出来讲），我们先看一从babel编译JSX返回的代码结构，可运行的最终代码（简化版）应该是这样的：

``` javascript
"use strict";

// 引入react
var _react = _interopRequireDefault(require("react"));

function _inherits(subClass, superClass) {
  // 重新构建subClass.prototype，将subClass.prototype上的constructor指向自身
  // 将subClass.__proto__指向superClass
}

function _createClass(Constructor, protoProps, staticProps) {
  // 挂载一般（public/protected/private/default）成员函数
  // 挂载static成员函数
}

function _createSuper(Derived) {
  // 返回父类构造函数
}

var MyComponent = (function(_React$Component) {
    // 处理下面的MyComponent，让它“继承”传入的参数
    _inherits(MyComponent, _React$Component);
  	
  	var _super = _createSuper(MyComponent);
  
    // 下面这个就是真正的class MyComponent中的内容了
  	// 由于函数提升，它会被提升到作用域的顶端
    function MyComponent() {
      var _this;
      
      // 执行父类构造函数
      _this = _super.call(this, props);
      
      // 完成在constructor中编写的内容
      // 在返回的对象上继续挂载初始化的数据
      _defineProperty(_this, "handleClick", function() {
        // ...
      })
      return _this;
    }
  
 		// 在构造器上挂载成员函数
    _createClass(MyComponent, [{
   		key: "render",
    	value: function render() {
      	return /*#__PURE__*/_react.default.createElement("div", {
        	onClick: this.handleClick
      	}, this.state.msg);
    	}
  	}]);
    
    return MyComponent;
})(_react.default.Component)
```

⬆️	以上就是浏览器可运行版的JavaScript代码（简化版）。

由此可见，我们在JSX模板中编写的html部分本质上就是`React.createElement`的语法糖，解析器最终将把`render`函数的返回值解析成`React.createElement("div")`。

进一步看，我们发现了代码黑盒的部分，也就是React做了处理的部分有两处，这可以引导我们进入React源码的世界：

1. 把`<div></div>`解析成`React.createElement("div")`
2. 把我们的构造函数“继承”了`React.Component`

## React.createElement会构造出什么样的对象？

通过查看源码，可以发现，`createElement`有三个参数：`type`（想要构造DOM元素的种类），`config`（该对象的`ref`、`key`等等我们开发过程常用到的配置项）和`children`（子节点）。`createElement`主要针对开发环境不合理的代码给出一些warning，并将`children`挂载到了`config.props`上。为了simplify代码，这里用的基本上都是面向过程的写法。

最后，createElement返回了一个ReactElement，继续查看源码：

``` javascript
const ReactElement = function(type, key, ref, self, source, owner, props) {
  const element = {
    // 该对象作为ReactElement的标识
    $$typeof: REACT_ELEMENT_TYPE,
    // 不用多说，一些ReactElement固有的属性，开发中常见
    type: type,
    key: key,
    ref: ref,
    props: props,

    // 记录创建这个ReactElement的组件
    // 先猜测是用于区分同一个ReactElement在不同组件中不同表现使用的
    _owner: owner,
  }
  
  return element;
}
```

可以看到，`ReactElement`其实只是一个简单对象，上面解构挂载了刚才我们提到了`config`和处理完的`config.props`之类的内容，仅此而已。

## React.Component作为组件的父类，其中做了些什么处理？

查看源码，可以看到`React.Component`中挂载了以下内容：

``` typescript
function Component(props, context, updater) {
  this.props = props;
  // context和updater牵涉到了其它模块，因此放到后续深入
  this.context = context;
  // ReactNoopUpdateQueue是一个只提供warning的空updater
  this.updater = updater || ReactNoopUpdateQueue;
  this.refs = {};
}

Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
}

Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};

function PureComponent() {
  // ...
}
```

我们看到，`Component`上会挂载`props`，`context`和`updater`这些成员变量，同时会在原型上挂载公用的`setState`和`forceUpdate`方法。从变量名上看到，`setState`和`forceUpdate`应该是内部维护了一个更新队列，调用某个组件内的`setState`时，它被加入了这个更新队列，按顺序更新。

如果把React比作一个IDE，把页面从JavaScript到渲染上屏的过程比作运行一个C++程序，那么上面这些代码相当于为render做好了一些静态的准备。从`ReactDOM.render`开始，后面的过程就好比这个C++程序真正进入了运行时。那么下面开始才是真正的重点：React如何帮助我们render以及update一个组件？

## React组件的创建与更新

在一个React项目的根目录处，我们通常会写下如下的代码：

``` typescript
// ...
ReactDOM.render(<App />, document.getElementById('app'));
// ...
```

这一行代码启动了react构造我们组件树，处理数据并渲染到屏幕的过程，查看它的源码：

``` typescript
function render(
	element: React$Element<any>,
  container: Container,
  callback?: Function,
) {
  return legacyRenderSubtreeIntoContainer(
    null,
    element,
    container,
    false,
    callback,
  );
}
```

我们使用React的初衷是为了“构建快速响应的用户界面”。所以我们最关注的就是React如何把静止的html“hydrate”成响应式的React组件，并能根据用户操作动态地更新。

从legacyRenderSubtreeIntoContainer函数开始，我们已经可以开始观察根结点是如何被hydrate成组件的：

``` typescript
function legacyRenderSubtreeIntoContainer(
	// ...
	container: Container
) {
  // 要被hydrate的节点
	let root: RootType = (container._reactRootContainer: any);
  // React Fiber链表的头节点
  let fiberRoot;
  if (!root) {
    // 开始hydrate
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    // 得到root的结果后，便得到React Fiber的头节点
    fiberRoot = root._internalRoot;
    // ...
  }
  // ...
}
```

沿着legacyCreateRootFromDOMContainer构造的链路，我们可以追踪到ReactDOMRoot.js，发现里面引入并调用的这两个正是创建与更新React组件的函数：

``` typescript
import {
  createContainer,
  updateContainer,
} from 'react-reconciler/src/ReactFiberReconciler';
```

我们在进入该函数前，首先需要补充关于`React reconciler`的知识，否则后面将很难读懂代码。

React中有许多功能，其中最核心的调度更新部分就是协调器（reconciler），它和渲染器（renderer）模块是独立的，并且都是可插拔的（pluginnable）。React所做的工作相当于一个编译器，它把开发者传递的数据修改指令翻译成可同步视图更新的代码，并交给机器运行。之所以没有说“交给浏览器主线程运行”，正是因为`reconciler`和`renderer`的可插拔性。React可以使用同一个`reconciler`和不同的`renderer`，让同一套代码运转在不同的平台上（硬件、VR、原生APP等）。

除了要明白`reconciler`在React中的位置，我们还需要知道React历史上的两套调度算法：

### stack reconciler

曾经（React 15及以前），React使用的是一套名为`stack reconciler`的调度算法。

如何理解`stack reconciler`？在计算机里，函数是一个子程序，计算机主要使用栈来存放函数调用过程中的数据（参数、返回地址等）。当层层的函数嵌套地调用时，栈会越来越深，而当被调用的函数接连执行完毕时，栈不断地弹出顶部的内容，除了得到了返回值以外，其它的状态又回到了函数调用前。

`stack reconciler`也是这样的思路。当我们开始构建/更新组件树时，更新组件的过程就是一个个子程序，不断地被压入栈中，在子程序执行完毕后才会接连弹出。而这个调度的过程是一个同步的过程，也就是说，当组件树很深时，一次更新的耗时也会相应提高。

众所周知，浏览器的渲染进程为了保证页面显示和脚本内的数据保持一致，同时包含了JavaScript解释器（一条线程）和渲染器（一条线程），而这两条线程又是互斥的。因此，当`stack reconciler`的一次更新非常庞大时，JavaScript线程会一直阻塞渲染器线程，使得页面没办法得到更新，体现出来就是掉帧和卡顿。

不同于后台服务，UI的更新速度会大大影响到用户的体验，稍慢几毫秒，用户所体验到的掉帧感觉就会非常明显，同时也会影响到交互。因此，React提出了一种新思路，实现一个`fiber`，主动地分片调度，使页面能及时得到更新，保证帧率不会降低得太严重。

### fiber

React 16版本以后，`React fiber`正式面世。`Fiber`并非React提出的新概念，理解`React fiber`，首先要理解`fiber`。

`Fiber`是在用户态下存在的，操作系统的内核态并不知道`fiber`的存在。`Fiber`是线程里更细粒度的存在，但就像`React fiber`一样，它是我们程序员自己实现的，并不是由操作系统调度的。在操作系统中，线程是由内核进行抢占式调度的，但`fiber`不同，它的调度算法是由我们程序员自己定义的，因此它会选择恰当的时机把操作权主动交还给线程（或者其他`fiber`）。

也许这个时候我们想到了`coroutine`，我们也可以这样来理解：`coroutine`是一种语言级别的构造（像go、kotlin里面就有`coroutine`），而`fiber`是`coroutine`的一种实现。

> Fibers describe essentially the same concept as coroutines. The distinction, if there is any, is that coroutines are a language-level construct, a form of control flow, while fibers are a systems-level construct, viewed as threads that happen to not run concurrently. It is contentious which of the two concepts has priority: fibers may be viewed as an implementation of coroutines, or as a substrate on which to implement coroutines.

以上也是wikipedia中它们区别的描述，当然它这里提到的`systems-level construct`表示的应该是windows中实现的`fiber`。像`React fiber`就并非是`systems-level`的。过多的我们不展开，还是要回到`React reconciler`。

## 系列文章

React源码学习讨论（二）：React reconciler调度算法（编写中……）

（更多……）