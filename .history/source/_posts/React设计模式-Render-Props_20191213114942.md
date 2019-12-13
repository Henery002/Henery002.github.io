---
title: React设计模式-Render Props
date: 2019-01-11 11:42:14
tags:
---

## 写在前面

React官方文档对render props做出的介绍是这样的：
> The term “render prop” refers to a technique for sharing code between React components using a prop whose value is a function.

本文将针对Render Props的使用方法，对比高阶组件，简单了解为什么需要使用以及如何正确使用这两种设计模式。

## 什么是Render Props

**render props** 是一种**在不重复代码的情况下实现组件共享**的方法。同为React的设计模式，render props与**高阶组件（HOC，Higher-Order Components）**的思想一样，都是为了组件复用以达到**DRY模式**。

一个栗子：
```javascript
<DataProvider render={data => (
  <h1>Hello {data.target}</h1>
)}>
```

通过使用props来定义呈现的内容，组件只起到注入功能，而不需要知道它如何应用于UI层。render props模式意味着用于通过定义单独的组件来传递props方法，来指示共享组件应该返回的内容。

Render Props的**核心思想**是：通过一个函数将class组件的state作为props传递给纯函数组件。
```javascript
import React from 'react';

const SharedComponent  extends React.Component {
  state = {...}
  render() {
    return (
      <div>
        {this.props.render(this.state)}
      </div>
    )
  }
}

export default SharedComponent;
```

this.props.render是由另一个组件传递过来的。为了使用 SharedComponent 组件，可以进行这样的操作：
```javascript
import React from 'react';
import SharedComponent from './SharedComponent';

const SayHello = () => {
  <SharedComponent render={(state) => (
    <span>Hello, {...state}</span>
  )} />
};
```

其中，{this.props.render(this.state)} 这个函数，将其state作为参数传入props.render方法中，调用时直接取组件所需的state即可。

**值得注意的是：**

> 渲染属性这种设计模式虽然叫作 render props，但是并不一定就要使用 render 这个名字来传递目标渲染函数，开发者可以修改成任意其他名称。

## 为什么需要Render Props

搜罗到一个通俗易懂的案例：

- **场景一：**
假设我们正在开发一个电子商务应用程序。它与其他电子商务应用程序一样，向用户显示所有可购买产品，并且用户可以将任何产品添加到购物车。我们将从 API 获取产品数据，并将产品目录显示为卡片列表。

我们一般会写一个这样的 ProductList 组件来处理这个需求：
```javascript
import React, { Component } from "react";

class ProductList extends Component {
  state = {
    products: []
  };

  componentDidMount() {
    getProducts().then(products => {
      this.setState({
        products
      });
    });
  }

  render() {
    return (
      <ul>
        {this.state.products.map(product => (
          <li>
            <span>{product.name}</span>
            <a href="#">Add to Cart</a>
          </li>
        ))}
      </ul>
    );
  }
}

export { ProductList };
```

- **场景二：**
在上一个场景中，对于管理员来说，有一个管理门户，他们可以在其中添加或删除产品。在此门户中，要从同一 API 获取产品数据，并以**表格形式**显示产品目录。
```javascript
import React, { Component } from "react";

class ProductTable extends Component {
  state = {
    products: []
  };

  componentDidMount() {
    getProducts().then(products => {
      this.setState({
        products
      });
    });
  }

  handleDelete = currentProduct => {
    const remainingProducts = this.state.products.filter(
      product => product.id !== currentProduct.id
    );
    deleteProducts(currentProduct.id).then(() => {
      this.setState({
        products: remainingProducts
      });
    });
  };

  render() {
    return (
      <table>
        <thead>
          <tr>
            <th>Product Name</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {this.state.products.map(product => (
            <tr key={product.id}>
              <td>{product.name}</td>
              <td>
                <button onClick={() => this.handleDelete(product)}>
                  Delete
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    );
  }
}

export { ProductTable };
```

现在我们来分析一下：

上面两个场景都实现了产品数据的获取。但是当出现下述情况时，上面写的两个组件将暴露出弊端：

1. 在不同场景下必须用不同的方式展示数据时；
2. 当改为从localStorage中访问数据而不是从API获取时；
3. 在产品目录表格中，需要使用具有不同操作的按钮而不都是删除按钮时
4. ......

如果每个场景下都手动写一个这个样的组件，那么将有大量冗余的代码。
从上面来看，**获取数据**和**显示数据**是两个独立的点，那么我们是不是可以据此分离出独立组件来处理这样的逻辑？

重构第一个组件 **ProductList**：
```javascript
import React from "react";

const ProductList = ({ products }) => {
  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          <span>{product.name}</span>
          <a href="#">Add to Cart</a>
        </li>
      ))}
    </ul>
  );
};

export { ProductList };
```

同理，第二个组件 ProductTable 接收产品数据作为属性，并把数据渲染到表格中。

现在来创建一个 ProductData 组件：
```javascript
import React, { Component } from "react";

class ProductData extends Component {
  state = {
    products: []
  };

  componentDidMount() {
    getProducts().then(products => {
      this.setState({
        products
      });
    });
  }

  render() {
    return {
      // 此处应该处理怎样的逻辑？
    };
  }
}

export { ProductData };
```

这个组件仍然是从API获取数据，逻辑同 *ProductList* 组件一样。但是在 render 方法中，却不再是调用 *ProductList* 组件。如果这个组件可以询问开发者需要渲染什么内容，那么问题就会到有效解决。比如在场景一中，我们告诉它要渲染 *ProductList* 组件，在场景二中让它渲染 *ProductTable* 组件。

对于一个组件，会询问它该在什么场景下渲染什么内容，这就是render props发挥作用的地方。这种设计思想进一步推动了代码的复用。

## 如何使用Render Props

从概念层面不难理解render props的原理，现在来看看以上面的需求为例，该如何应用渲染属性。

首先我们要**封装一个组件 *ProductData*** （还是上面的案例），通过属性传递给该组件一个函数，用来获取产品数据（公共方法），并将这些数据提供给以属性方式传递进来的函数。这个传递进来的函数就可以对产品数据做任何场景需求上的操作了。
import React, { Component } from "react";

// ProductData组件
class ProductData extends Component {
  state = {
    products: []
  };

  componentDidMount() {
    getProducts().then(products => {
      this.setState({
        products
      });
    });
  }

  render() {
    return this.props.render({
      products: this.state.products
    });
  }
}

export { ProductData };

就像前面说的，有了这个组件，我们就需要在不同场景下使用它：
// 场景一
<ProductData
  render={({ products }) => <ProductList products={products} />}
/>

// 场景二
<ProductData
  render={({ products }) => <ProductTable products={products} />}
/>

// 场景三
<ProductData render={({ products }) => (
  <h1>
      Number of Products:
      <strong>{products.length}</strong>
  </h1>
)} />

这就是render props用来构建高度可复用组件的设计思想，它已经成为了一种通用的设计模式。正如之前分享的一篇文章《React组件解耦》中说的，渲染属性和高阶函数可以起到同样的作用，甚至可以相互转化。
HOC设计模式
限于篇幅的原因，这里简要引入一下高阶组件的设计思想。
高阶组件本质是一个函数，接收一个组件作为其参数，然后返回一个组件，同时为该组件添加一些额外的功能（有点类似于ES6中的装饰器函数）。

封装一个高阶函数组件：
// 函数式组件，接收一个组件作为参数，并返回一个新组件
function withProductData(WrappedComponent, selectData){
  return class extends React.Component {
    state = {
      products: []
    };

    componentDidMount() {
      getProducts().then(products => {
        this.setState({
          products
        });
      });
    }

    render() {
      return (
        <WrappedComponent
          // 使用最新的数据渲染组件
          products={this.state.products}
          // 将已有的props属性传递给原组件
          {...this.props}
        />
      );
    }
  };
}

在高阶组件中数据的获取和状态更新的原理与render props一样。只是高阶组件中组件类位于函数内部。这个函数接收一个组件作为参数，然后在内部的render方法中渲染这个组件，并且添加额外的属性。
// 在不同场景下调用高阶组件
const ProductListWithData = withProductData(ProductList);
const ProductTableWithData = withProductData(ProductTable);

// ...
render() {
  return (
    <div>
      <ProductListWithData />
      <ProductTableWithData />
    </div
  )
}
Render Props Vs HOC
高阶组件优劣并存：
支持ES6；
复用性强，HOC是纯函数且返回值还是组件，在使用时可以多层嵌套，在不同情境下使用特定的HOC组合也便于调试；
由于是纯函数，支持传入多个参数，增强了其适用范围。

当有多个HOC一同使用时，无法直接判断子组件的props是哪个HOC负责传递的；
重复命名的问题：若父子组件有同样名称的props，或使用的多个HOC中存在相同名称的props，则存在覆盖问题，而且react也不会报错；
HOC的嵌套产生了很多无用的组件，加深了组件层级；
使用静态构建。在上例中，当 ProductListWithData 被创建时，调用了一次 withProductData 中的静态构建。而在 render 中调用构建方法才是react所倡导的动态构建思想。此外，在 render 中构建还可以更好地利用react的生命周期。 

render props相较而言，有这些优势：
同样支持ES6；
不用担心props的命名问题，在render函数中只取所需要的state即可；
不会产生过多的无用空组件加深层级；
使用动态构建。即所有的改变都在 render 中触发，可以更好地利用react的生命周期。
结语
同样作为React的设计模式，个人认为还没有达到谁必须要取代谁的地步。有很多第三方开源库也都使用了这两种设计模式来设计API的实现。在实际开发项目中，根据业务需要，选择一种适合自己的方式即可。

附：使用render props的开源库
参考文章
segmentfault - 《render props从介绍到实践》- Anx
React官方文档 - 《Render Props》
《Understanding React JS Render Props》- Richard Kotze
《Higher-order components Vs Render Props》- Richard Kotze
掘金 - 《高阶组件的那些事》- Srtian
简书 - 《React组件Render Props VS HOC设计模式》- Perkin


