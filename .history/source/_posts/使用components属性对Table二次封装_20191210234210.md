---
title: 使用components属性对Table二次封装
date: 2019-03-20 10:00
tags:
---

应用场景
需求：
DanaStudio v4.2中数据治理 - 标准化：实现两个表格间行数据的拖拽联动效果

效果图：
图片: https://uploader.shimo.im/f/MY3pr7nzbC02KSHr.gif
图一
由Table拖拽排序带来的思考
一些尝试
最初曾尝试通过对Table组件的 columns 属性进行包裹，以期达到封装 DragSource 和 DropTarget 的效果，但后来发现，antd根部不会识别这样的修改。
那么antd到底是否开放了一些自定义事件供开发者使用呢？

官方案例
antd官方文档提供了Table单一表格的行拖拽排序，并没有暴露出更多可详细配置的API。
在拖拽排序的案例中，BodyRow既是DragSource，也是DopTarget。那么既然如此，理应也允许针对BodyRow进行二次封装，达到DragSource与DropTarget的独立，从而实现跨表格的行（row）或列（column）的拖拽。

在开始本文的探讨之前，先来回顾一下react dnd的基础知识。
React DnD回顾
什么是react dnd
React DnD是一组React高阶组件，它可以帮开发者构建复杂的拖放接口，同时保持组件解耦。借助react dnd的拖动事件在组件之间传输数据，组件根据拖放事件改变它们的外观和状态（数据）。

核心API
react-dnd使用的核心API有以下几个：
DragSource：用于包装你需要拖动的组件，使组件能够被拖拽。
DropTarget：用于包装接收拖拽元素的组件，使组件能够放置。
DragDropContext：用于包装拖拽根组件，DragSource 和 DropTarget 都需要包裹在DragDropContext内。
DragDropContextProvider：与 DragDropContex 类似，用DragDropContextProvider元素包裹拖拽根组件。

API参数
@DragSource(type, spec, collect)
@DropTarget(types, spec, collect)
DragSource、DropTarget分别有三个参数：
type: 拖拽类型，必填。
spec：拖拽事件的方法对象，必填。
collect：拖拽过程中需要注入组件的props信息，接收两个参数 connect 和 monitor，必填。

其中：
type
当source组件的type和target组件的type一致时，target组件可以接受source组件。
其值类型可以使string， symbol，也可以是一个函数来返回该组件的其他props。

spec
spec是用来定义特定方法的一个对象，如source组件的spec可以定义拖动相关的事件，target组件的spec可以定义放置相关的事件。具体如下：

DragSource speObj：
beginDrag(props, monitor, component)拖动开始时触发的事件，required。返回与props相关的对象。
endDrag(props, monitor, component)拖动结束时触发的事件，optional。
canDrag(props, monitor)当前是否可以拖拽的事件，optional。
isDrag(props, monitor)拖拽时触发的事件，optional。
// DragSource源组件的 spec 属性
const dragSpec = {
  beginDrag(props) {
    console.log('%c beginDrag...', 'color: #F8B400', props);
  },
  endDrag(props, monitor) {
    const item = monitor.getItem();
    const dropResult = monitor.getDropResult();
    console.log(item, dropResult, 'endDrag...');
  },
};

DropTarget specObj：
drop(props, monitor, component)组件被放下时触发的事件，optional。
hover(props, monitor, component)组件在DropTarget上方时响应的事件，optional。
canDrop(props, monitor)组件可以被放置时触发的事件，可选。
// DropTarget目标组件
const targetSpec = {
  drop(props, monitor) {
    // dropTarget 行的值
    const { record } = props.children[0].props;
    // dragSource 行的值
    const dragItem = monitor.getItem();
  },
};

其中，specObj这个对象方法中的相关参数为：
props：组件当前的props。
monitor：查询当前的拖拽状态，比如当前拖拽的item和他的type，当前拖拽的offsets，当前是否dropped。
component：当前组件的实例。

collect
collect是一个函数，默认有两个参数：connect 和 monitor。该函数将返回一个对象，这个对象会被注入到props中，也就是说，我们可以在目标组件中通过 this.props 获取到通过collect注入进来的所有属性。
// TODO 封装 DropTarget 组件
const DropBodyRow = DropTarget(Types.CARD, targetSpec, (connect, monitor) => {
  return {
    // 由 collect 函数返回的对象中的API，可以在组件中利用 this.props 获取
    connectDropTarget: connect.dropTarget(),
    isOver: monitor.isOver(),
    canDrop: monitor.canDrop(),
  };
})(BodyRow);

// 在组件中通过使用 this.props 获取API
class BodyRow extends PureComponent {
  render() {
    const { connectDropTarget, isOver, canDrop } = this.props;

    return connectDropTarget(
      <tr {...restProps} />
    );
  }
}

在 collect 函数的两个参数中：
connect
source组件  的 collect 中的 connect 是DragSourceConnector 的实例，它内置了两个方法：dragSource() 和 dragPreview() 。dragSource() 返回一个方法，将source组件传入这个方法，可以将 source DOM 和 React DnD backend 连接起来；dragPreview() 返回一个方法，可以将其传入节点，作为拖拽预览时的角色。
target组件  的 collect 中 connect 是 DropTargetConnector 的实例。内置的方法 dropTarget() 对应 DragSource()，返回可以将 drop target 和 React DnD backend 连接起来的方法。
monitormonitor用来查询当前的拖拽状态，其对应实例内置了很多方法。由于使用场景不是很必要，这里就不再一一赘述。详情可以移步官网研究。

在熟悉了React DnD的相关内容后，来看看如何针对Table的BodyRow进行二次封装。

对Table Row二次封装
antd3.0 新增了 components 属性，用于覆盖默认的table元素。官方更新日志上也只是提供了简单的配置说明，没有详细的案例。

antd源码解读
因为官方文档并没有针对这个属性做过多解释，所以就找到了它的源码来看一哈。
在antd table的源码中发现，Table组件在初始化时会调用 createComponents 方法。

图片: https://uploader.shimo.im/f/U409TsQht4YQn9Vr.png
图二

这个方法最终在 this.components 上设置了一个 components 对象。
图片: https://uploader.shimo.im/f/EuiJRAWDakMycp3z.png
图三

this.components 最终在 render() 犯法中被调用。

图片: https://uploader.shimo.im/f/RtoEralutDARNcAA.png
图四

定位到 components 属性，不难发现，最终用来覆盖修改 row 或者 column 的，就是它。
图片: https://uploader.shimo.im/f/N3cXkCYTHP4oq3zP.png
图五

TableComponent 属性重新定义了表格的 header、body，其中，在我们的场景中所需要的就是 row 属性，将自己用 DropTarget 二次封装好的 BodyRow 组件挂到components中。components属性最终将会在render()方法中被调用，实现预期效果。
图片: https://uploader.shimo.im/f/5jZ6PljW5iULIU7A.png
图六

拓展案例
Table行拖拽排序
Table列拖拽排序
Table 能否提供拖拽事件或者开放自定义事件

参考文章
React DnD官方文档
React DnD的使用
antd3.0中新增的 components 属性的使用
