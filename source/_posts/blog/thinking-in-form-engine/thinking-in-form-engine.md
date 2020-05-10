---
title: 表单引擎设计思想
author: 大大白
date: 2018-06-28 16:48:00
categories:
- WEB前端
- 设计思想
tags: 
- 表单引擎
---

本文介绍的表单引擎是一个采用了单向数据流，双向数据绑定，三大事件机制，四大核心驱动，打通五脏六腑七经八脉，按照九大设计模式为指导思想的十全十美的表单引擎。本文将带你走进表单引擎的世界，为你阐述其设计思想，让你领悟其设计精髓！
<!-- more -->

## 设计思想
根据配置信息生成表单。配置信息配置了表单的布局样式，交互方式，以及关联关系。
根据配置信息在组件库中找到对应的组件，在运行时进行实例化。
借鉴spring的思想通过控制反转的方式将组件都放在context中管理。组件之间如果需要通信可以通过引擎触发，引擎充当一个中介者的角色来处理这些信息。
一些装饰器可以增强组件的能力。
容器组件事先定义好了子组件的布局方式。

### 控制反转（Inversion of Control）
表单引擎将组件`component`注入到`IoC`容器里面，并通过组件的`key`进行查找。这里使用了`依赖注入（Dependency Injection）`和`依赖查找（Dependency Lookup）`的思想。并通过单例模式的方式让组件在容器中保持唯一。
依赖注入和依赖查找是控制反转（Inversion of Control，缩写为IoC）的重要思想，简单来说就是把原来自己的控制权限转交给别人。
依赖注入（Dependency Injection）：将组件注入到容器。
依赖查找（Dependency Lookup）：在容器中找到组件。

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

### 面向切面（Aspect-Oriented Programming）
表单引擎在预编译的时候把`this._onChange`与`this._aop`绑定在一起，在`onChange`运行时的时候才真正去出发`this._aop`中的方法。
面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。
预编译（precompile）：编译时把当前的方法放入AOP中。
运行时（runtime）：运行时AOP中会动态的把需要的方法织入。
```js
 <RadioGroup options={options}
             onChange={this._aop.bindArgs(this._onChange, this, { target: { value: !this.state.value } })}
             value={this.state.value} />
```

### 数据驱动与事件驱动
数据驱动，是按照redux的数据流去渲染。事件驱动，则是按照触发的事件通过中介者进行调度。


```js
const config = {
    AOP: true
}
let eventConfig = {
    source: 'StandardCase.EditMode',//源组件
    target: 'Rate.EditMode',//目标组件
    sourceFunc: 'onChange',//源方法
    targetFunc: 'onChange',//目标方法dynamicFunc
    targetFuncArgs: [ '4' ],//参数
    triggerMode: 'custom',//触发方式：static静态触发，按照预设参数去触发；dynamic动态触发，在运行时根据触发的方法带过来的参数触发；custom自定义的，按照自定义的规则触发。
    DSL: '(function (args){if(args===true){return 4} else {return 2}})'//自定义方法
}
```

## 规范
组件必须装饰@FormEngine，必须继承BaseComponent。
组件必须以函数驱动为导向，出入格式要规范参数全部用数组传递等等。
每个组件都必须提供自己的API列表，方便其他开发者配置和使用。

### 基类BaseComponent
每个组件把自己转交给IoC容器，然后继承BaseComponent在切面中编写代码。
实例化IoC容器，并且把组件注入到容器中。
绑定aop切面，让子类可以方便的使用aop切面。

```javascript
export class BaseComponent extends Component {
    static displayName = 'BaseComponent'
    constructor(props) {
        super(props)
        this.IoC = IoC.getInstance()
        this._aop = bind(this.aop, this)
    }
    componentWillMount() {
        if (this.props.displayName && this.props.displayName !== 'undefined') {
            if (!(this.props.displayName.indexOf('FormEngine') > -1)) {
                this.IoC.injection(this.props.displayName, this)
            }
        }
    }
    aop(...args) {
        return this.props.AOP(...args)
    }
}
```

### 核心类FormEngine
为什么要使用反向继承？
从宏观的角度来看。继承子组件的属性，并且可以控制子组件。
从微观的角度来看。继承了子组件就可以把AOP传给子组件，子组件可以使用AOP进行切面绑定。但是这个作用，属性代理也可以实现。只有反向继承子组件，才能使用子组件继承自BaseComponent的IoC容器，然后再通过IoC容器控制其他组件。针对这一点，属性代理无法实现。如果不反向继承，这将无法获得IoC容器。

使用反向继承
不使用this.IoC，而是使用this.props.IoC可不可以？
不可以。这里的this指向EC，不是指向子组件。EC的props里面并没有IoC容器，EC的IoC容器是继承自子组件子组件继承自BaseComponent的IoC容器。

this.IoC和this.props.IoC有什么区别？
反向继承了子组件子组件继承了BaseComponent，所以EC是有this.IoC的。但是，这里的this指向的是EC并不是指向子组件，所以this.props.IoC并没有值。


使用属性代理
this.IoC和this.props.IoC有什么区别？
属性代理没有继承子组件所以没有this.IoC，同时也没有人传递给他props，所以这里的this.props.IoC也是undefined


综上所述。
这里一定要用反向继承，并且一定要用this.IoC来获取容器以此来控制组件。

```js
export function FormEngine(WrappedComponent) {
    let ProxyComponent = new Proxy(WrappedComponent, handler)
    return class EC extends WrappedComponent {
        static displayName = `FormEngine(${getDisplayName(WrappedComponent)})`
        constructor() {
            super()
        }
        //如果需要对子组件的方法进行切面，需要显示调用此方法。
        //注意这里的this是EC组件return之前的WrappedComponent，所以这里的this指向WrappedComponent但是它（参考对象是EC组件）没有props的。
        //this2是子组件的this，指向真实的子组件，所以有props，它与BaseComponent类似都是经历了React生命周期后形成的。
        AOP(func, self, ...args) {
            //在类的加载、实例化的时期执行。
            // return () => {
            //在类的运行时期执行。
            if (config.AOP) {
                if (this.displayName === eventConfig.source) {
                    if (func.name && func.name.indexOf(eventConfig.sourceFunc) > -1) {
                        let toolInstance = this.IoC.lookup(eventConfig.target)
                        if (toolInstance && typeof toolInstance[eventConfig.targetFunc] === 'function') {
                            if (eventConfig.triggerMode === 'static') {//static静态触发，按照预设参数去触发
                                let event = args.pop()//触发的事件
                                toolInstance[eventConfig.targetFunc](...eventConfig.targetFuncArgs)//触发目标组件方法
                                func(event)//触发源组件方法
                            } else if (eventConfig.triggerMode === 'dynamic') {//dynamic动态触发，在运行时根据触发的方法带过来的参数触发
                                let params = args[0]
                                toolInstance[eventConfig.targetFunc](params)
                                func(params)
                            } else if (eventConfig.triggerMode === 'custom') {//custom自定义的，按照自定义的规则触发
                                let event = args.pop()//触发的事件
                                let inData = event.target.value
                                let customFunc = eval(eventConfig.DSL)
                                let outData = customFunc(inData)
                                toolInstance[eventConfig.targetFunc](outData)
                                func(event)
                            }
                        }
                    }
                }
            } else {
                func(...args)
            }
            // }
        }
        render() {
            const props = Object.assign({}, this.props, { AOP: this.AOP, IoC: this.IoC, displayName: WrappedComponent.displayName })
            return <ProxyComponent {...props} />
        }
    }
}
function getDisplayName(WrappedComponent) {
    return WrappedComponent.displayName || WrappedComponent.name || 'Component'
}
let handler = {
    construct: function (target, args) {
        // console.log(`construct ${target}!`)
        return Reflect.construct(target, args)
    },
    get: function (target, key, receiver) {
        // console.log(`getting ${key}!`)
        return Reflect.get(target, key, receiver)
    },
    set: function (target, key, value, receiver) {
        // console.log(`setting ${key}!`)
        return Reflect.set(target, key, value, receiver)
    }
}

```

## 如何接入
加入@FormEngine注解，把自己注入到IoC容器中。
继承BaseComponent，可以使用IoC容器和AOP切面。

```js
@FormEngine
export default class EditMode extends BaseComponent {
    static displayName = 'Rate.EditMode'
    static propTypes = {
        submitData: PropTypes.func
    }
    constructor(props) {
        super(props)
        this.state = {
            value: 1
        }
        this._onChange = bind(this.onChange, this)
        this._confirm = bind(this.confirm, this)
    }
    confirm() {
        this.props.submitData(this.state.value)
    }
    onChange(args) {
        this.setState({
            value: args
        })
    }
    dynamicFunc(args) {
        this.setState({
            value: 3
        })
    }
    render() {
        return (
            <div className={styles.container}>
                <div>
                    <Rate allowHalf
                        allowClear
                        value={this.state.value}
                        defaultValue={1}
                        onChange={this._onChange} />
                </div>
                <Button className={styles.confirmBtn}
                    htmlType='button'
                    loading={false}
                    shape={null}
                    icon={null}
                    onClick={this._aop.bindArgs(this._confirm, this)}
                    size='default'
                    type='primary'
                    ghost>
                    确定
                </Button>
            </div>
        )
    }
}
```

```js
@FormEngine
export default class EditMode extends BaseComponent {
    static displayName = 'StandardCase.EditMode'
    static propTypes = {
        preview: PropTypes.bool,
        toolData: PropTypes.bool,
        submitData: PropTypes.func,
        AOP: PropTypes.func
    }
    constructor(props) {
        super(props)
        this.state = {
            value: true
        }
        this._onChange = bind(this.onChange, this)
        this._confirm = bind(this.confirm, this)
    }
    onChange(e) {
        this.setState({
            value: e.target.value
        })
    }
    confirm() {
        this.props.submitData(this.state.value)
    }
    render() {
        const options = [
            { label: '标准例', value: true },
            { label: '反例', value: false }
        ]
        return (
            <div className={styles.container}>
                <div>
                    <RadioGroup options={options}
                        onChange={this._aop.bindArgs(this._onChange, this, { target: { value: !this.state.value } })}
                        value={this.state.value} />
                </div>
                <Button className={styles.confirmBtn}
                    htmlType='button'
                    loading={false}
                    shape={null}
                    icon={null}
                    onClick={this._confirm}
                    size='default'
                    type='primary'
                    ghost>
                    确定
                </Button>
            </div>
        )
    }
}
```

ToolsMng类
```js
export default new class ToolsMng {
	constructor() {
		this.toolMap = {}
	}
	registerTool(toolKey, tool) {
		this.toolMap[toolKey] = tool
	}
	getTool(toolKey) {
		return this.toolMap[toolKey]
	}
	getToolMap() {
		return this.toolMap
	}
}
```

ToolsInit类
```js
import toolMng from './toolsMng'
toolMng.registerTool('audio_recorder', require('./audioRecorder'))
```

具体创建方式
```js
const comp = toolMng.getTool(tool.key, tool.version)//取出配置信息的key
React.createElement(comp || ToolNotFound, props)
```
