---
title: React Hooks
date: 2018-11-29 13:39:57
tags:
---

## 写在前面

在开始本文的探讨之前，先来看看 React 官网 对 Hooks 做出的介绍：

> Hooks are a new feature proposal that lets you use state and other React features without writing a class. They’re currently in React v16.7.0-alpha and ...

## 什么是 Hooks

Hooks 是 React 函数组件内一类特殊的函数，通常以“use”开头，如“useState”、“useEffect”。Hooks 使得开发者能够在 function component 里继续使用 state 和 life-style，以及使用 custom hook 复用业务逻辑。“Hooks” 的本意是“钩子”，在 React 中，hooks 作为一系列特殊的函数，使函数式组件内部能够“钩住” React 内部的 state 和 life-style。

React Hooks API 向后兼容，它在解决了一些既有问题的情况下，不仅使开发者能够更好地使用 state 和 life-cycles，真正强大的功能点在于更加轻松地复用组件逻辑，也就是下文将要讲述的 custom hooks。

## Hooks 可以做什么

React 中函数式组件是一个 pure render component，没有 state 和 component life-cycle。如果开发者需要使用 state 和 life-cycle，就需要将 function component 改写成 class component。在既有的 React API 下，这个模式有一些缺点：

- **组件间通信的耦合度高，组件树臃肿**
  在既有的模式下，React 组件间通信有两种模式：

1. 单向数据流
2. 通过 Redux 的 global store 实现全局状态和各组件间的解耦

当有些状态不适合放在 global store 中时，组件间逻辑的复用和通信就变得很困难（只能一层一层往下传）。这一点在高阶组件（HOC, Higher-Order Components）和渲染属性（Render Props）中更为常见。为了复用一些业务逻辑，有时候会单独编写一些高阶组件用来向下传递状态。这样就会导致当开发者的业务规模变得越来越庞大的时候，一些无关 UI 的 wrapper 组件越来越多，React 组件树就会变得愈加臃肿庞杂。例如，在有些时候，一个简单的 Tooltip component 里面都潜藏了很多层额外的组件，使得开发和调试的效率变得低下。

在新的 React Hook 中，开发者可以创建自定义 Hook（Custom Hook），用以复用一些逻辑，这些逻辑会成为一个独立的逻辑单元，不再出现在组件树中，但仍然能够响应 React 在渲染之间的变化。

- **由 js 的 class 带来的疑惑**

简单来说，就是 js 语法中关于 this 的指向以及原型链、继承这类问题常常会给新手开发者学习 React 带来困惑。比如，在 React 组件内的事件监听之前需要手动绑定 this 的问题。

为了解决这些问题，Hooks 允许开发者在没有 class 的情况下使用 React 的 features。

总结来说，推出 Hooks 的动机是解决长时间使用和维护 react 过程中遇到的一些难以避免的问题，如：

1. 难以重用和共享组件中的与状态相关的逻辑
2. 逻辑复杂的组件难以开发与维护，当我们的组件需要处理多个互不相关的 local state 时，每个生命周期函数中可能会包含着各种互不相关的逻辑在里面。
3. 类组件中的 this 增加学习成本，类组件在基于现有工具的优化上存在些许问题。
4. 由于业务变动，函数组件不得不改为类组件等等。

## Hooks 的规则

Hooks 仍然是 Javascript 函数，但是在使用时必须遵循两个常规 js 函数不具备的规则：

- 只能在顶层调用 Hooks，不要在循环语句、条件判断或嵌套函数中调用 Hooks
- 仅从 React 函数式组件中调用 Hooks，不要从常规 js 函数中调用 Hooks
- 开发者也可以在 custom hooks 中调用 Hooks。

  遵循了上述规则之后，就能够确保每次组件渲染时都以相同的顺序调用 Hook。

为此，官方提供了一个名叫 [eslint-plugin-react-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks) 的 eslint 插件用来强制执行上述规则。

1. 将 eslint-plugin-react-hooks 安装到项目中：

`npm install eslint-plugin-react-hooks@next --save-dev`

2. 配置 eslint

```json
{
  "plugins": [
    // ...
    "react-hooks"
  ],
  "rules": {
    // ...
    "react-hooks/rules-of-hooks": "error"
  }
}
```

## Hooks API

由于 Hooks 是**向后兼容**（Backwards compatiblity, 也叫向下兼容）的，class component 不会被移除，向后兼容的友好性使得开发者可以将项目逐渐迁移到新的 Hooks API。

Hooks API 主要分为三种：

1. **State hooks**：在函数式组件中使用 state
2. **Effect hooks**：在函数式组件中使用 life-cycle 和 side effect
3. **Custom hooks**：用来复用组件逻辑，解决了上述动机中阐述的第一个问题

- **State hooks**

  看一个例子：

```javascript
import { useState } from "react";

function Example() {
  // TODO 声明一个变量名为 count 的 state
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

在这个例子中，“useState”就是一个 hook，通过它我们可以嵌入组件内部的 state，这个 useState 函数可以返回一个 pair 元组，其中第一个值是对应当前 hook 的 state 值，第二个是定义修改这个 state 的方法。

其实这个 pair 的两个返回值分别对应的就是 React 中 hooks 之前的 this.state、this.setState。

那么我们的组件中不可能只用到一个 state 和一个 setState，所以，useState 这个 hook 可以在一个函数组件中多次使用：

```javascript
function ExampleWithManyStates() {
  // 声明多个state变量
  const [age, setAge] = useState(28);
  const [fruit, setFruit] = useState("apple");
  const [todos, setTodos] = useState([{ text: "React Hooks" }]);
  // ...
}
```

与以前在一个 class component 中只能写一个 state 相比，hooks 就避免了组件的 state 结构过于臃肿，每个 state 能够被单独处理。另外，useState 的写法可读性高，用户一眼就可以看出和 state 相关的两个变量，如上面例子中的[age, setAge]。

- **Effect hooks**
-

首先了解下什么是 _Side Effect_（副作用），副作用是指函数或表达式的行为依赖于外部环境：

- 函数或表达式修改了它的 scope 之外的状态
- 函数或表达式除了返回语句之外，还与外部环境或它所调用的函数有交互行为

Effect hook 为函数式组件添加了执行 side effects 的能力。我们使用 useEffect 这个 API 来处理副作用：

```javascript
import { useState, useEffect } from "react";

function Example() {
  const [count, setCount] = useState(0);

  // 类似于 class component 中的 componentDidMount 和 componentDidUpdate:
  useEffect(() => {
    // 调用浏览器 API 更改文档标题 title
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

React Hooks 解决了在函数式组件中使用 life-cycle 的问题。useEffect Hook 可以看作是 React 类生命周期方法中 componentDidMount、componentDidUpdate 和 componentWillMount 的组合，这个钩子函数类似于 redux 中的 subscrib，每当 React 因为 state 或 props 改变重新 render 之后，就会触发 useEffect 里的回调函数。useEffect 的代码既会在初始化时执行，也会在后续每次 rerender 时执行。

- **Custom hooks**

构建自定义钩子函数可以将组件逻辑提取到可重用组件中，即在 hooks 中引用其它 hooks。下面这个官方给出的 demo 演示了一个聊天程序中朋友是否处于在线状态：

```javascript
import { useState, useEffect } from "react";

// 底层 Hooks, 返回布尔值：是否在线
function useFriendStatusBoolean(props) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friendID, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friendID, handleStatusChange);
    };
  });

  return isOnline;
}

// 上层 Hooks，根据在线状态返回字符串：Loading... or Online or Offline
function useFriendStatusString(props) {
  const isOnline = useFriendStatusBoolean(props.friend.id);

  if (isOnline === null) {
    return "Loading...";
  }
  return isOnline ? "Online" : "Offline";
}

// 使用了底层 Hooks 的 UI
function FriendListItem(props) {
  const isOnline = useFriendStatusBoolean(props.friend.id);

  return (
    <li style={{ color: isOnline ? "green" : "black" }}>{props.friend.name}</li>
  );
}

// 使用了上层 Hooks 的 UI
function FriendListStatus(props) {
  const statu = useFriendStatusString(props.friend.id);

  return <li>{statu}</li>;
}
```

在这个例子中有两个 Hooks：**useFriendStatusBoolean** 与 **useFriendStatusString**，**useFriendStatusString** 是利用 **useFriendStatusBoolean** 生成的新 Hook，因为这两个 Hooks 的数据是联动的，所以可以提供给两个不同的 UI 组件使用：**FriendListItem** 与 **FriendListStatus**，两个 UI 组件的状态也将是联动的。

值得注意的是：

1. 这里，**useFriendStatusBoolean 和 useFriendStatusString 是有状态组件**（使用 useState），没有渲染，所以就可以作为 Custom Hooks 被任何 UI 组件所调用，实现了**组件的复用**。
2. **FriendListItem 和 FriendListStatus 是有渲染的组件**（返回了 JSX），没有状态（没有使用 useState），所以他们就是一个纯 UI 组件。

- **Other Built-in Hooks API**

除了上述介绍的 useState、useEffect 之外，还有一些内置的其它 hooks，如：

- useContext：替代了<Context.Consumer>使用 render props 的写法，使组件树更加简洁。
- useReducer：相当于组件自带的 redux reducer，负责接收 dipatch 分发的 action 并更新 state。
- useCallback
- ......

## Hooks FAQ

参见官方文档针对 React Hooks 一些常见问题的解答：
https://react.css88.com/docs/hooks-faq.html

## 参考文章

1. [React 中文文档 - Hooks 概述](https://react.css88.com/docs/hooks-intro.html)
2. [Hooks FAQ](https://react.css88.com/docs/hooks-faq.html)
3. [精读《React Hooks》](https://juejin.im/post/5be8d3def265da611a476231)
4. [像呼吸一样自然：React Hooks + RxJS](https://zhuanlan.zhihu.com/p/50921147?utm_source=wechat_timeline&utm_medium=social&utm_oi=27826472353792&from=timeline&isappinstalled=0)
5. [对 React Hooks 的一些思考](https://zhuanlan.zhihu.com/p/48264713)
