title: 前端优化个人分享
speaker: 张淑峰
url: https://github.com/ksky521/nodePPT
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes

[slide]
## 今天要讲的内容
----
* 何为卡顿？
* React造成卡顿的原因
* 一个简单的优化demo
  * 优化方案
  * ImmutableJs
  * 结论
* 讨论dva架构下的优化方案

[slide]
# 何为卡顿
* js的执行时间过长，导致页面暂时无法执行其他操作，仿佛有个断点
  * 原因1：js存在死循环或万次级别的循环
  * 原因2：过多的React虚拟节点在进行比较操作（即在执行render方法）
* 由于浏览器行为产生的重绘过慢
  * 增加、删除和修改可见DOM元素
  * 页面初始化的渲染
  * 移动DOM元素
  * 修改CSS样式，改变DOM元素的尺寸
  * DOM元素内容改变，使得尺寸被撑大
  * 浏览器窗口尺寸改变
  * 浏览器窗口滚动
[slide]
```我们讨论如何优化js，而重绘过慢问题暂不讨论[尤其是庞大的dom数无法变更的前提下]```
[slide]
## React的操作dom的机制

<img src="/imgs/react.png"/>

* 性能消耗比较：<br/>
```虚拟DOM与DOM之间的比较更新 >> App与虚拟DOM的比较更新```

* 优化的原则：增加App与虚拟DOM的比较以减少虚拟DOM与DOM之间的比较更新

[slide]
## 讨论大数据table优化问题
<img src="/imgs/Table.png"/>

[slide]
## ImmutableJs是什么

ImmutableJs是facebook提供的一个扩充JS数据类型的工具，它提供了不同数据类型的深比较。

[slide]

## 这里为什么要用ImmutableJs


- 用于简化sholdUpdateComponent的代码，如果不使用ImmutableJs，我们将在不同的待优化的组件写入不同规则的判断逻辑(和传入的数据有关)。

- 使用ImmutableJs，我们可以写一个修饰器，该修饰器可以往被修饰的组件中写入同样逻辑的sholdUpdateComponent。

[slide]
## 结论
- 优化建议1：将节点数过多的业务性组件[<span class='text-danger'>与数据直接交流</span>]拆分为若干个非业务性子组件，子组件是否重新渲染完全决定于传入的props是否改变；

- 优化建议2：重写非业务线组件的shouldComponentUpdate，提高比较级别，以减少组件重新render的概率；

- 优化建议3：由于对象的比较规则多种多样，建议引入ImmutableJS进行统一的对象比较。

[slide]
# dva架构下的优化

dva架构使用redux来管理数据层，即建议我们把state统一抽到store里来进行管理，对dva架构的优化分为三种：
* <span class="text-info">维护一套合理、整洁、不相互影响的数据极其维护方案</span>
* <span class="text-info">使用一套流畅、不重叠的数据传递方式</span>
* <span class="text-info">被传递数据的组件提高传参比较级别以减少rerender的次数</span>

[slide]
## 维护一套合理、整洁、不相互影响的数据及其维护方案
错误的示例1：
<pre><code class="javascript">
// 无分组显示
effects: {
  ...
  *noGroupView({ payload }, { select, put, call }) {
    // removeGroupType 和 setGroup之间从业务上讲并没有先后关系
    // 此类代码增加了state的修改次数，从而增加了render次数
    yield put({
      type: 'removeGroupType',
    })
    
    yield put({
      type: 'setGroup',
      payload: {
        isGroup: false,
      }
    })
  },
  ...
}


==== 等同于 ===
this.setState(newState1, () => { this.setState(newState2) })

</code></pre>

[slide]
### 临时优化方案1
<pre><code>
effects: {
  *noGroupView({ payload }, { select, put, call }) {
    // effect调用每个reducer都附带一个参数，来确定这个操作是否会重新渲染
    yield put({
      type: 'removeGroupType',
      payload: {
        isReRender: false
      }
    })
    yield put({
      type: 'setGroup',
      payload: {
        isGroup: false,
      }
    })
  },
}
=== 等同于 ===
this.state = ....;
this.setState(newState)

</code></pre>

[slide]
错误示例2：
<pre><code>
*queryBefore({ payload }, { call, put, select }) {
  yield put({ type: 'setCurrentQuery', payload: { ...payload } })
  yield put({ type: 'queryAlertList' })
  // 一个model调用其他model的reducer
  // 这样会导致代码的阅读性很差
  // 让一个model的数据维护方法不再单纯
  yield put({ type: 'alertDetail/removeGroupType' })
},
</code></pre>
优化方案2：
<pre><code>
...
*queryBefore({ payload }, { call, put, select }) {
  yield put({ type: 'setCurrentQuery', payload: { ...payload, isReRender: false } })
  yield put({ type: 'queryAlertList' })
  // 提供一个effects方法的回调方法，
  payload && payload.afterQueryBefore && payload.afterQueryBefore();
},

...
// 这种写法是比较好解释的
dispatch({
  type: 'xxx/queryBefore',
  payload: {
    afterQueryBefore: () => {
      dispatch({ type: 'alertDetail/removeGroupType' })
    }
  }
})
</code></pre>

[slide]
## 使用一套流畅、不重叠的数据传递方式

错误示例1：父类和子类connect了同样的数据（父类和子类都是业务组件）
<pre><code>
class Father {
  render() {
    return (
      < Child />
    )
  }
}
export default conÏnect((state) => { 
  return { p1: state.p1 } 
})(Father)

....

class Child {
  ...
}

export default connect((state) => { 
  return {p1: state.p1 } 
})(Child)
</code></pre>
[slide]
修改原则：在需要redux store中某些数据的时候先看看父级是否已经获取，如果是则直接传递

<pre><code>
class Father {
  render() {
    return (
      < Child p1={ this.props.p1 }/>
    )
  }
}
export default connect((state) => { 
  return { p1: state.p1 } 
})(Father)

....

class Child {
  ...
}

export default Child
</code></pre>

[slide]
## 被传递数据的组件提高传参比较级别以减少rerender的次数
- 这种优化方案尤其适用于一些批量渲染的内容，比如说Table的每一个Tr；
- 最好的情况是替代redux提供的是否重新渲染判断

[slide]
## redux的connect机制
<pre><code>
export default function connect(mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {
   return function wrapWithConnect(WrappedComponent) {
     class Connect extends Component {
           constructor(props, context) {
               // 从祖先Component处获得store
               this.store = props.store || context.store
               this.stateProps = computeStateProps(this.store, props)
               this.dispatchProps = computeDispatchProps(this.store, props)
               this.state = { storeState: null }
               // 对stateProps、dispatchProps、parentProps进行合并      
               this.updateState()
           }
           shouldComponentUpdate(nextProps, nextState) {
               // 进行判断，当数据发生改变时，Component重新渲染
               if (propsChanged || mapStateProducedChange || dispatchPropsChanged) {
                 this.updateState(nextProps)
                 return true
               }
           }
           componentDidMount() {
               // 改变Component的state
             this.store.subscribe(() = {
                 this.setState({
                   storeState: this.store.getState()
                 })
             })
           }
           render() {
               // 生成包裹组件Connect
             return (
               <WrappedComponent {...this.nextState} />
             )
           }
       }
       Connect.contextTypes = {
         store: storeShape
       }
       return Connect;
   }
}
</code></pre>
[slide]
> 我们遇到的麻烦是redux进行的比较是浅比较，两个新的对象即使内容一样，但是比较的结果还是不一致

-----

> 比较理想的情况是为每一个组件提供统一的shouldComponentUpdate方法，通过props和state的深比较来决定是否重新渲染

[slide]
* 如何提供浏览器重绘效率
> 现代浏览器（谷歌、ie等）已经大大提高了重绘效率，我们已经很难从代码层面提高重绘效率；
故剩下来的方法便是尽可能减少显示中的节点数；
[slide]
<h1>隐藏的样式</h1>
> <code type="css"> 1. opacity: 0</code> 
<br/>
节点未消失，只是透明度为0，浏览器依旧需要为此花费精力重绘；
<br/>
> <code type="css"> 2. display: none</code>
<br/>
节点消失，浏览器不需要为此花费精力重绘
[slide]
<h1>优化的空间</h1>
> 现在的一些组件库，在实现隐藏的效果的时候倾向于使用<code type="css">opacity: 0</code>
<br/>
> 例如antd的<strong>Tab组件</strong>

[slide]
# 谢谢大家