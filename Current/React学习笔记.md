## React学习笔记

- React的基础概念与使用模式

  - **组件化与Dom打包**，思路与Flutter十分接近——构造一个组件，声明组件的绘制方式（实际上是对Dom元素的集合封装，配上类似小程序的数据变量声明方式），以及该组件的实际使用位置（这里也是写成标签形式，总体行为类似Android的Java声明View，而不是Flutter中的直接标签嵌套。）

  - 上面的组件化实现基础来自于**JSX语法**，形式类似DataBinding或者小程序，阻止了表现层与渲染层的分离，使单个组件可以以可拆分的单位（文件）独自发挥作用。

  - 基于**Virtual DOM**实现的差分渲染。和Flutter以及Vue一样，在state发生变化时，以差分比较的方式更新对应的渲染节点。

  - 例：

    ```javascript
    import React, { Component } from 'react';
    import { render } from 'react-dom';
    
    class HelloMessage extends Component {
      render() {
        return <div>Hello {this.props.name}</div>;
      }
    }
    
    // 加载组件到 DOM 元素 mountNode
    // 注意这里和上面的标签都是直接做变量使用的 不需要引号或者大括号
    render(<HelloMessage name="John" />, mountNode);
    ```

- JSX写起来的时候基本上和JS相差不大，但是由于二次编译，会与JS存在一些关键字冲突和转义，这里允许通过`dangerouslySetInnerHTML={XXX}`来写无转义的HTML片段，并使用部分用于代替的关键字（如for->htmlFor）

- 组件状态的声明与变更像极了Flutter，包括无状态和有状态组件的定义：

  ```javascript
  import React, { Component } from 'react';
  import { render } from 'react-dom';
  
  class LikeButton extends Component {
    constructor(props) {
      super(props);
      this.state = { liked: false };
    }
  
    handleClick(e) {
      this.setState({ liked: !this.state.liked });
    }
  
    render() {
      const text = this.state.liked ? 'like' : 'haven\'t liked';
      return (
        <p onClick={this.handleClick.bind(this)}>
            You {text} this. Click to toggle.
        </p>
      );
    }
  }
  
  render(
      <LikeButton />,
      document.getElementById('example')
  );
  ```

- React的**交互事件处理**大多使用React API中的事件（如onClick等），这些事件成为合成事件，经过中间层的特别处理；而通过`[add/remove]EventListener`方法附加的事件监听称为原生事件，和浏览器本身的事件相同。如果存在这两种事件的混用，可能会因为合成事件的处理逻辑产生一些预料外的情况，比如阻止事件冒泡传递失败（原生事件）、在异步操作中获取事件等（合成事件）。

- React允许直接操作DOM元素，可以通过Refs标记和`findDOMNode`方法来获取DOM元素对象。

  ```javascript
  <input
            ref="theInput"
            value={this.state.userInput}
            onChange={this.handleChange.bind(this)}
          />
  ```

- 循环插入组件时，需要为每个子组件设置一个唯一的key（为了差分更新的实现），且这些key不能是整数值（最好是一个带前缀的字符串），否则会被再次渲染时执行排序操作。

- **DataFlow**相当于是React数据管理与更新的一套架构模板，在React本身的View组合和State机制之外，封装了存储和管理的相关功能。主要有官方的Flux和Redux两种实现。

  - Flux应用主要分为四个部分

    ```
    the dispatcher //类似Presenter和Model交互的部分
    
    处理动作分发，维护 Store 之间的依赖关系
    
    the stores //类似Model以及Presenter处理源数据的部分，不暴露原始数据，输入全部来自于订阅的Action
    
    数据和逻辑部分
    
    the views //View的交互响应部分
    
    React 组件，这一层可以看作 controller-views，作为视图同时响应用户交互
    
    the actions //时间实体类，类似MotionEvent
    
    提供给 dispatcher 传递数据给 store
    ```

  - Flux本身还是基于观察者模式，需要在Store和ActionCreator中依赖Dispatcher主动触发事件分发和事件处理（Store处理完之后还会再通过Dispatcher去通知订阅的View，当然具体的订阅分发代码还是写在Action里的），通过订阅关系联系起各个组件发挥功能。

  - Redux在此基础上更进一步，它将ActionCreator和Store中关于事件分发的代码去除，将前者视为单一的数据提供者，后者变成了一个给以恒定输入就会恒定输出的状态机，或者说**纯函数（pure function）**，以此完成与Dispatcher的解耦。

    除此之外，原来保存在Store中的状态量现在也不再显式声明，其功能只提供到根据输入状态输出对应状态为止，不再关注最后提供给View的实际状态。

    > 这里的感觉就是，Redux将原本掺杂了MVP模式下P和M功能的Store进行了简化，变成了纯粹的逻辑处理工具。

