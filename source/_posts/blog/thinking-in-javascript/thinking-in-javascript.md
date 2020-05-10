---
title: Javascript设计思想
author: 大大白
date: 2020-03-20 12:00:00
categories:
- WEB前端
- 设计思想
tags: 
- Javascript设计思想
---
设计模式，我觉得应该分成设计和模式两个东西来看。设计有一些原则，是前辈们不断日积月累总结出的理论，通过对这些理论的综合实践又产生了二十几种的模式。
就像java的反射机制一样，javascript也有它独有的特性，那就是原型与原型链、作用域及闭包、异步和单线程。通过结合这些独有的特性所设计实现的代码才能真正体现出javascript的设计模式。

<!-- more -->


## 设计原则
设计原则这个比较通用，无论是什么编程语言，他们想要描述的最佳实践的思想是比较类似的，所以这部分的概念是比较统一的。

### 基本概念
#### 单一职责原则（Single Responsibility Principle）
一个类应该只有一个发生变化的原因。类的复杂性降低，可读性提高。类、函数和接口都要要遵循单一职责原则。遵守单一职责原则，将不同的职责封装到不同的类或模块中。
#### 开闭原则（Open Close Principle）
一个软件实体，如类、模块和函数应该对扩展开放，对修改关闭。稳定，灵活，可扩展。
#### 里氏替换原则（Liskov Substitution Principle）
所有引用基类的地方必须能透明地使用其子类的对象。用父类定义，用子类赋值。超类存在的地方，子类是可以替换的。子类继承了父类的方法，同时子类还可以扩展自己的方法。面向对象的语言的三大特点是继承、封装、多态，里氏替换原则就是依赖于继承、多态这两大特性。
#### 接口隔离原则（Interface Segregation Principle）
客户端不应该依赖它不需要的接口。应当为客户端提供尽可能小的单独的接口，而不是提供大的总的接口。使用多个专门的接口比使用单一的总接口要好。
#### 迪米特法则（Law Of Demeter）
又叫最少知识原则，一个软件实体应当尽可能少的与其他实体发生相互作用。只与你的直接朋友交谈，不跟“陌生人”说话。
#### 依赖倒置原则（Dependence Inversion Principle）
实现尽量依赖抽象，不依赖具体实现。上层模块不应该依赖底层模块，它们都应该依赖于抽象。抽象不应该依赖于细节，细节应该依赖于抽象。
#### 组合/聚合复用原则（Composite/Aggregate Reuse Principle CARP）
尽量使用合成/聚合达到复用，尽量少用继承。一个类中有另一个类的对象。对比继承子类会被父类污染。但是如果是非常明确的一定必须要有的，继承依然可以使用。组合比较灵活。

### 我的理解
#### 单一职责原则（Single Responsibility Principle）
有一个需求用户输入email之后可以点击＋在添加一个email输入框继续添加。这个可以做成两个独立的组件，一个email，一个appendContainer。他们的职责很简单，emial可以用来提供用户输入email还有输入框的基本样式。而appendContainer则用来复制创建子元素，并把数据准确的传递给指定的元素。
比如常用的radio组件，一个是button，另一个是group。其实很简单就是一个选中操作，但是经常会有需求出现一个group中只能选中一个radio，在这里会分成两个组件。一个是radio很简单就是一个render组件，另一个是控制子节点选中的状态，也很简单用来维护状态就行了。

#### 开闭原则（Open Close Principle）
在外部可以很方面的修改item的渲染方式，看是要做成左右结构还是上下结构。这个符合开闭原则。
组件通信，字段之间的关联关系。通过dependencies，中介者模式。引擎把组件之间需要互相调用的所有数据都传递出去给中介者，用户可以在中介者里面修改组件如何通讯。比如说一个评分组件可以评1~5分，1分和5分要说明原因所以会出现一个textarea的组件，2-4分不用说明原因。那么引擎不能把这个逻辑直接实现，而是把组件之间的状态都传递出去，给用户自己去实现。不然将来万一要变就会导致需要重新修改引擎，所以引擎把重心放在组件的数据状态传递出去，给用户自己实现，因为需要控制的权限已经在IoC容器里面，完全可以达到目的。

#### 里氏替换原则（Liskov Substitution Principle）
父类定义子类赋值。用BaseContainer定义了变量，在具体实现的时候指定了具体的TableContainer，ListContainer，AppendContainer。
容器类，都需要渲染this.props.children，但是不容的容器对children渲染方式都不一样，比如List容器，Tabs容器。因此我们定义了baseContainer基类实现了渲染children的方法，然后ListContainer和TabsContainer去继承这个基类，定义自己的渲染方式。基类只是把children渲染出来，子类则是定义了渲染后如何展示。

#### 接口隔离原则（Interface Segregation Principle）
和服务端接口交互尽量小而单独，而不是大而总的接口。tab组件有3个列表待使用，待审核，已完成。点击按钮之后跳转到第2个列表的第1项。那这时候应该是提供两个接口，一个是切换tab的，另一个是选中列表项的，而不是一个接口全部搞定。
比如tab切换后要选中第1项，那么我会提供两个接口来给外部调用，一个是切换tab，另一个是选中第1项。

#### 迪米特法则（Law Of Demeter）
最少知道原则。在组件通信的时候引擎认为只要把这些组件运行时的状态和数据抛出去就行了，剩下的由开发人员去做，引擎不用关心怎么做，只要做好自己的事情就好了。
dependencies从开发人员的角度出发是开闭原则，从引擎的角度出发是迪米特法则。还有类似的比如状态模式。我只要能访问调用它的方法就好了，至于它里面是怎么做的我并不关心。
#### 依赖倒置原则（Dependence Inversion Principle）
List组件里面的item在实现的时候应该依赖于抽象，而不是具体的把item实现成某个布局。这个符合依赖倒置。
List容器依赖抽象实现，在map循环数组的时候把renderItem放给外面，给外面定义，本身只是抽象的调用了renderItem方法表示在这个位置会有一个Item。因为Item可能会有上下结构，也可能会有左右结构，可能会有图片按钮，这些要放给外面去实现。而不能具体的把这个item就直接实现成左右或者上下结构，这样当用户需要修改的时候就无法修改变成重新复制一份出来修改。



## 模式
### 基本概念
#### 创建型模式
这些设计模式提供了一种在创建对象的同时隐藏创建逻辑的方式，而不是使用 new 运算符直接实例化对象。这使得程序在判断针对某个给定实例需要创建哪些对象时更加灵活。	
工厂模式（Factory Pattern）
抽象工厂模式（Abstract Factory Pattern）
单例模式（Singleton Pattern）
建造者模式（Builder Pattern）
原型模式（Prototype Pattern）

#### 结构型模式
这些设计模式关注类和对象的组合。继承的概念被用来组合接口和定义组合对象获得新功能的方式。	
适配器模式（Adapter Pattern）
桥接模式（Bridge Pattern）
过滤器模式（Filter、Criteria Pattern）
组合模式（Composite Pattern）
装饰器模式（Decorator Pattern）
外观模式（Facade Pattern）
享元模式（Flyweight Pattern）
代理模式（Proxy Pattern）

#### 行为型模式
这些设计模式特别关注对象之间的通信。	
责任链模式（Chain of Responsibility Pattern）
命令模式（Command Pattern）
解释器模式（Interpreter Pattern）
迭代器模式（Iterator Pattern）
中介者模式（Mediator Pattern）
备忘录模式（Memento Pattern）
观察者模式（Observer Pattern）
状态模式（State Pattern）
空对象模式（Null Object Pattern）
策略模式（Strategy Pattern）
模板模式（Template Pattern）
访问者模式（Visitor Pattern）


### 我的理解
#### 观察者模式和发布订阅模式
他们最大的区别就是发布订阅模式有个事件调度中心，他们的联系是发布订阅模式是观察者模式的一种实现，发布订阅模式是广义上的观察者模式。
```js
// pubSub.js
import emitter from 'utils/eventUtil'
/**
 * 发布订阅模式
 * 可以用来处理setState，但是不能用来处理调用对象的方法。这个还需完善，可以考虑用IOC容器。
 * @param {*} subscriberKey
 */
function PubSub(subscriberKey) {
  return WrappedComponent => {
    return class extends WrappedComponent {
      constructor(props) {
        super(props)
      }
      componentDidMount() {
        super.componentDidMount && super.componentDidMount()
        this.eventEmitter = emitter.addListener(subscriberKey, this.apiSetState)
      }
      apiSetState = ({ key, value }) => {
        super.setState({
          [key]: value,
        })
      }
      componentWillUnmount() {
        super.componentWillUnmount && super.componentWillUnmount()
        emitter.removeListener(subscriberKey, this.apiSetState)
      }
    }
  }
}
export default PubSub

// eventUtil.js
import { EventEmitter } from 'events'
export default new EventEmitter()

// sidebar.js
@PubSub('SideBar.setState')
export default class SideBar extends Component {
  constructor(){
    this.state={
      xxx:false
    }
  }
}

// folderItem.js
import emitter from 'utils/eventUtil'
export default class FolderItem extends Component {
  selectedDocType = () => {
      emitter.emit('SideBar.setState', {key: 'xxx', value: false})
  }
}
```

#### 代理模式与面向切面
面向切面编程其实就是代理模式
表单引擎在预编译的时候把this._onChange与this._aop绑定在一起，在onChange运行时的时候才真正去触发this._aop中的方法。这就相当于是一个表达式还没有真正执行。
面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。每个组件把自己转交给IoC容器，然后继承BaseComponent在切面中编写代码。实例化IoC容器，并且把组件注入到容器中。绑定aop切面，让子类可以方便的使用aop切面。

```javascript
//BaseComponent包含了IoC容器，还有一个用于拦截组件行为的方法
export class BaseComponent extends Component {
    static displayName = 'BaseComponent'
    constructor(props) {
        super(props)
        this.IoC = IoC.getInstance()
    }
    componentDidMount() {
        if (this.props.displayName && this.props.displayName !== 'undefined') {
            if (!(this.props.displayName.indexOf('FormEngine') > -1)) {
                this.IoC.injection(this.props.displayName, this)
            }
        }
    }
    aop = (...args) => {
        return this.props.AOP(...args)
    }
}

export default class A extends BaseComponent {
    constructor(props) {
        super(props)
        this.state = {
            value: 1
        }
    }
    confirm = () => {
        this.props.submitData(this.state.value)
    }
    render() {
        return (<div onClick={this._aop.bindArgs(this._confirm, this)}>xxx</div>)
    }
}
```

使用者无权访问目标对象，中间加代理，通过代理做权限和控制。就类似AOP切面编程。
适配器提供不同的接口；代理模式提供一模一样的接口。
代理模式显示原有功能但是经过限制或者阉割之后的；装饰器扩展功能原有功能不变不冲突可以直接使用
```js
class ReadImg{
  constructor(fileName){
    this.fileName = fileName
    this.loadFromDisk()
  }
  display(){
    console.log('display...',this.fileName)
  }
  loadFormDisk(){
    console.log('loading...',this.fileName)
  }
}
class ProxyImg {
  construtor(fileName){
    this.realImg = new RedImg(fileName)
  }
  display(){
    this.realImg.display()
  }
}
```

#### 访问者模式与控制反转
控制反转就是把原来自己控制的权限转交给外部控制的过程叫做控制反转，也叫控制转移。
依赖注入就是把控制权转交出去，依赖查找就是外部容器控制这个对象。
控制反转其实和中介者模式比较像。组件之间不再互相通信互相影响，而是通过一个中介者去调配处理这些组件之间的关系。
这是为了解决跨多层级组件之间的通讯的问题。组件的数据没有在 redux 中维护，而是在 state 中。组件内部特定的方法。

场景：文章列表组件 Articles，他们归属于同一个分类。点击分类按钮，打开侧边栏分类列表组件 Categories，分类列表顶部有一个搜索框 Input 用来搜索分类，点击分类 CategoryItem 后刷新文章列表。

需求：选中某个分类，关闭侧边栏，清空 Input 框。这要求 Articles、Input 的数据都要从 redux 存取。当选中 CategoryItem 后，同时设置 Articles 和 Input 的数据。但是 Input 只不过是一个普通的输入框却要绕一大圈到 redux 里面，如果 Input 不在 redux 存储，这个需求将很难完成。
这时候 IoC 体现出了价值，它可以把 Categories 放入 IoC 容器中，在 CategoryItem 里面去调用 Categories 的方法来清空 Input。因为在实际开发中 Input 受控方式直接在 state 是最方便的。

总的来说，适用于以下这种场景。由 state 控制的，或者是组件内部的方法，没办法通过 callback，或者修改 redux 就可以实现的。

比如一个 List 组件，里面的数据已经是一个完整的对象，当打开单个 Item 的时候不需要再请求一次，因此这个 Item 的 data 经常会被放在 state 维护。但是当 Modal 弹出框后需要修改这个 data，问题就出现了。因为你虽然更新了 redux，但是当前 modal 里面的数据却是在父容器的 state 中，所以这时候无法和新数据保持一致。当然这种情况可以用 callback 的方式解决，就是当更新完数据之后调用一个 callback 把最新的值 set 回来。

getData()、renderItem()调用外部的方法。实现的方式依赖于抽象而不是具体。

```js
class IoC {
    constructor() {
        this.context = {}
    }
    static getInstance() {
        if (!this.instance) {
            this.instance = new IoC()
        }
        return this.instance
    }
    injection(key, component) {
        this.context[key] = component
    }
    lookup(key) {
        return this.context[key]
    }
}
```

```js
// 容器
const ctx = {};
// 把组件注入到容器
function CTRL(subscriberKey) {
  return WrappedComponent => {
    return class extends WrappedComponent {
      constructor(props) {
        super(props);
      }
      componentDidMount() {
        super.componentDidMount && super.componentDidMount();
        ctx[subscriberKey] = this;
      }

      componentWillUnmount() {
        super.componentWillUnmount && super.componentWillUnmount();
      }
    };
  };
}

export { CTRL, ctx };
```

#### 单例模式
由于所有的组件都要互相通讯，所以必须保证所有的组件都在同一个容器中，所以使用单例模式创建IoC容器。
```js
function Sequence() {
  if (!Sequence.instance) {
    var privateIndex = 0;
    this.next = () => {
      return ++privateIndex;
    };
    Sequence.instance = this;
  }
  return Sequence.instance;
}
```

#### 原型模式
`Object.create()`
对比js中的prototype，prototype是es6 class底层实现的原理
Object.create()和new Object()的区别
```js
let prototype = {
  getName: function () {
    return this.first + ' ' + this.last
  }
  say: function(){
    alert('hello')
  }
}
let x = Object.create(prototype)
x.first = 'A'
x.last = 'B'
```

#### 模块模式
充分利用了闭包的特性

```js
var Counter = (function() {
  var privateCounter = 0;
  function changeBy(val) {
    privateCounter += val;
  }
  return {
    increment: function() {
      changeBy(1);
    },
    decrement: function() {
      changeBy(-1);
    },
    value: function() {
      return privateCounter;
    }
  }   
})();

console.log(Counter.value()); /* logs 0 */
Counter.increment();
Counter.increment();
console.log(Counter.value()); /* logs 2 */
Counter.decrement();
console.log(Counter.value()); /* logs 1 */
```

```js
var makeCounter = function() {
  var privateCounter = 0;
  function changeBy(val) {
    privateCounter += val;
  }
  return {
    increment: function() {
      changeBy(1);
    },
    decrement: function() {
      changeBy(-1);
    },
    value: function() {
      return privateCounter;
    }
  }  
};

var Counter1 = makeCounter();
var Counter2 = makeCounter();
console.log(Counter1.value()); /* logs 0 */
Counter1.increment();
Counter1.increment();
console.log(Counter1.value()); /* logs 2 */
Counter1.decrement();
console.log(Counter1.value()); /* logs 1 */
console.log(Counter2.value()); /* logs 0 */
```

#### 状态模式
一个对象有状态变化
每个状态变化都有很多逻辑处理

有限状态机javascript-state-machine
promise就是一个状态模式
promise基本特性
1. promise是一个class
2. promise初始化的时候要传入函数
3. 传入的函数要有两个参数一个resolve另一个是reject
4. 实现.then方法有两个参数，成功了执行第一个参数，失败了执行第二个参数

promis三种状态：pending fullfilled rejected
pending->fullfilled 或者pending->rejected
不可逆向变化

```js
import StateMachine from 'javascript-state-machine'

let fsm = new StateMachine({
  init: 'pending',//初始化状态
  transitions:[
    {
      name:'resolve',
      from:'pending',
      to:'fullfilled'
    },
    {
      name:'reject',
      from:'pending',
      to:'rejected'
    }
  ]
  methods: {
    onResolve: function(state,data){
      // state 当前状态机实例；data fsm.resolve(xxx)
    }，
    onReject: function(state, data){
      // state 当前状态机实例；data fsm.reject(xxx)
    }
  }
})

// 定义promise
class MyPromise{
  constructor(fn){
    this.successList=[]
    this.failList=[]
    fn(function(){
      //resolve函数
      fsm.resolve(this)
    },function(){
      //reject函数
      fsm.reject(this)
    })
  }
  then(successFn,failFn){
    this.successList.push(successFn)
    this.failList.push(failFn)
  }
}
function loadImg(src){
  const promise = new Promise(function(resolve,reject){
    let img=document.createElement('img')
    img.onload=function(){
      resolve(img)
    }
    img.onerror=function(){
      reject()
    }
    img.src=src
  })
}
let src=''
let result = loadImg(src)
result.then(function(){
  console.log('ok1')
},function(){
  console.log('fail1')
})
result.then(function(){
  console.log('ok2')
},function(){
  console.log('fail2')
})
```

#### 模板模式
封装一个统一对外的方法
并发或顺序执行
```js
class Action{
  handle(){
    handle1()
    handle2()
    handle3()
  }
  handle1(){
    console.log(1)
  }
  handle2(){
    console.log(2)
  }
  handle3(){
    console.log(3)
  }
}
```

#### 命令模式
document.execCommand('bold')
document.execCommand('undo')
```js
class Receiver(){
  exec(){
    console.log('执行')
  }
}
// 命令者
class Command(){
  constructor(receiver){
    this.receiver=receiver
  }
  cmd(){
    console.log('执行命令')
    this.receiver.exec()
  }
}
class Invoker(){
  constructor(command){
    this.command = command
  }
  invoke(){
    console.log('开始')
    this.command.cmd()
  }
}
```

#### 备忘录模式
随时记录一个对象的状态变化，随时恢复之前的某个功能
```js
class Memento{
  constructor(content){
    this.conetnt = content
  }
  getContent(){
    return this.content
  }
}
class CareTaker{
  constructor(){
    this.list = []
  }
  add(mement){
    this.list.push(mement)
  }
  get(i){
    return this.list[i]
  }
}
class Editor{
  constructor(){
    this.content=null
  }
  setContent(content){
    this.content=content
  }
  getContent(){
    return this.content
  }
  saveContentToMementto(){
    return new Memento(this.content)
  }
  getcontentFromMemeto(memento){
    this.content = memento.getContent()
  }
}

```


#### 其他待整理
迭代器模式
顺序访问一个集合
使用者无需知道集合的内部结构
ES6 Iterator

装饰器模式。
就类似高阶函数。core-decorator封装了大量的装饰器。高阶函数，返回函数的闭包。

中介者模式。
通过dependencies把状态传出去。

外观模式
注意：不符合S I，需要谨慎使用，不可滥用。

桥接模式
实现和抽象分离
符合开闭原则

组合模式
vnode的实现方式

享元模式
共享元数据
共享内存（主要考虑内存，而非效率）
相同的数据，共享使用
无限下拉列表，将事件代理到高层节点，不然列表元素要是很多的话会绑定很多事件占用很多内存。像menu，tab这些组件都是这个思想

策略模式
不同的策略分开处理，避免出现大量的if...else或者switch...case

职责链模式
顺序执行,promise.then链式操作

访问者模式
将数据操作和数据结构进行分离

解释器模式
描述语言语法如何定义，如何解释和编译
