### 前言

`Vue.extend`是 Vue 里的一个全局 API，它提供了一种灵活的挂载组件的方式，这个 API 在日常开发中很少使用，毕竟只在碰到某些特殊的需求时它才能派上用场，但是我们也要学习它，学习这个 API 可以让我们对 Vue 更加了解，更加熟悉 Vue 的组件初始化和挂载流程，除此之外，也经常会有面试官问到这个东西。下面我们就来从源码到应用彻彻底底的看一看这个 API。

### Vue.extend 定义

引用一个官方的定义：

`使用基础 Vue 构造器，创建一个“子类”。参数是一个包含组件选项的对象。`

data 选项是特例，需要注意 - 在 Vue.extend() 中它必须是函数

```html
<div id="mount-point"></div>
```

```js
// 创建构造器
var Profile = Vue.extend({
  template: '<p>{{firstName}} {{lastName}} aka {{alias}}</p>',
  data: function() {
    return {
      firstName: 'Walter',
      lastName: 'White',
      alias: 'Heisenberg'
    }
  }
})
// 创建 Profile 实例，并挂载到一个元素上。
new Profile().$mount('#mount-point')
```

结果如下

```html
<p>Walter White aka Heisenberg</p>
```

> 之前没用过这个 API 的小伙伴看到这个定义肯定会一头雾水，但是没关系，相信你看完这篇文章后以定会理解的。

这个 API 可以实现很灵活的功能，比如 ElementUI 里的`$message`，我们使用`this.$message('hello')`的时候，其实就是通过这种方式创建一个组件实例，然后再将这个组件挂载到了 body 上，本篇文章也会分析如何实现这个组件，下面我们先来看下`Vue.extend`的源码，从根源上来了解它。

### 源码分析

你可以在源码目录`src/core/global-api/extend.js`下找到这个函数的定义

```js
export function initExtend(Vue: GlobalAPI) {
  // 这个cid是一个全局唯一的递增的id
  // 缓存的时候会用到它
  Vue.cid = 0
  let cid = 1

  /**
   * Class inheritance
   */
  Vue.extend = function(extendOptions: Object): Function {
    // extendOptions就是我我们传入的组件options
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    // 每次创建完Sub构造函数后，都会把这个函数储存在extendOptions上的_Ctor中
    // 下次如果用再同一个extendOptions创建Sub时
    // 就会直接从_Ctor返回
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }

    const name = extendOptions.name || Super.options.name
    if (process.env.NODE_ENV !== 'production' && name) {
      validateComponentName(name)
    }

    // 创建Sub构造函数
    const Sub = function VueComponent(options) {
      this._init(options)
    }

    // 继承Super，如果使用Vue.extend，这里的Super就是Vue
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid++

    // 将组件的options和Vue的options合并，得到一个完整的options
    // 可以理解为将Vue的一些全局的属性，比如全局注册的组件和mixin，分给了Sub
    Sub.options = mergeOptions(Super.options, extendOptions)
    Sub['super'] = Super

    // 下面两个设置了下代理，
    // 将props和computed代理到了原型上
    // 你可以不用关心这个
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    // 继承Vue的global-api
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // 继承assets的api，比如注册组件，指令，过滤器
    ASSET_TYPES.forEach(function(type) {
      Sub[type] = Super[type]
    })

    // 在components里添加一个自己
    // 不是主要逻辑，可以先不管
    if (name) {
      Sub.options.components[name] = Sub
    }

    // 将这些options保存起来
    // 一会创建实例的时候会用到
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // 设置缓存
    // 就是上文的缓存
    cachedCtors[SuperId] = Sub
    return Sub
  }
}

function initProps(Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}

function initComputed(Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}
```

其实这个`Vue.extend`做的事情很简单，就是继承 Vue，正如定义中说的那样，创建一个子类，最终返回的这个 Sub 是：

```js
const Sub = function VueComponent(options) {
  this._init(options)
}
```

那么上文的例子中的`new Profile()`执行的就是这个方法了，因为继承了 Vue 的原型，这里的`_init`就是 Vue 原型上的`_init`方法，你可以在源码目录下`src/core/instance/init.js`中找到它：

```js
Vue.prototype._init = function(options?: Object) {
  const vm: Component = this
  // a uid
  vm._uid = uid++

  let startTag, endTag
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    startTag = `vue-perf-start:${vm._uid}`
    endTag = `vue-perf-end:${vm._uid}`
    mark(startTag)
  }

  vm._isVue = true
  // merge options
  if (options && options._isComponent) {
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    initProxy(vm)
  } else {
    vm._renderProxy = vm
  }
  // expose real self
  vm._self = vmnext
  initLifecycle(vm)
  initEvents(vm)
  initRender(vm)
  callHook(vm, 'beforeCreate')
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')

  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    vm._name = formatComponentName(vm, false)
    mark(endTag)
    measure(`vue ${vm._name} init`, startTag, endTag)
  }

  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

这个函数里有很多逻辑，它主要做的事情就是初始化组件的事件，状态等，大多不是我们本次分析的重点，你目前只需要关心里面的这一段代码：

```js
if (options && options._isComponent) {
  initInternalComponent(vm, options)
} else {
  vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
  )
}
```

执行`new Profile()`的时候没有传任何参数，所以这里的 options 是 undefined，会走到 else 分值，然后`resolveConstructorOptions(vm.constructor)`其实就是拿到`Sub.options`这个东西，你可以在上文的`Vue.extend`源码中找到它，然后将`Sub.options`和`new Profile()`传入的`options`合并，再赋值给实例的`$options`，所以如果`new Profile()`的时候传入了一个 options，这个 options 将会合并到`vm.$options`上，然后在这个`_init`函数的最后判断了下`vm.$options.el`是否存在，存在的话就执行`vm.$mount`将组件挂载到 el 上，因为我们没有传 options，所以这里的 el 肯定是不存在的，所以你才会看到例子中的`new Profile().$mount('#mount-point')`手动执行了`$mount`方法，其实经过这些分析你就会发现，我们直接执行`new Profile({ el: '#mount-point' })`也是可以的，除了 el 也可以传其他参数，接着往下看就知道了。

> `$mount` 方法会执行“挂载”，其实内部的整个过程是很复杂的，会执行 render、update、patch 等等，由于这些不是本次文章的重点，你只需要知道她会将组件的 dom 挂载到对应的 dom 节点上就行了，如`$mount('#mount-point')`会把组件 dom 挂载到`#mount-point`这个元素上。

### 如何使用

经过上面的分析，你应该大致了解了`Vue.extend`的原理以及初始化过程，以及简单的使用，其实这个初始化和平时的`new Vue()`是一样的，毕竟两个执行的同一个方法。但是在实际的使用中，我们可能还需要给组件传 props，slots 以及绑定事件，下面我们来看下如何做到这些事情。

#### 使用 props

比如我们有一个 MessageBox 组件：

```html
<template>
  <div class="message-box">
    {{ message }}
  </div>
</template>

<script>
  export default {
    props: {
      message: {
        type: String,
        default: ''
      }
    }
  }
</script>
```

它需要一个 props 来显示这个`message`，在使用`Vue.extend`时，要想给组件传参数，我们需要在实例化的时候传一个 propsData，
如:

```js
const MessageBoxCtor = Vue.extend(MessageBox)
new MessageBox({
  propsData: {
    message: 'hello'
  }
}).$mount('#target')
```

你可能会不明白为什么要穿`propsData`，没关系，接下来就来搞懂它，毕竟文章的目的就是彻底分析。

在上文的`_init`函数中，在合并完`$options`后，还执行了一个函数`initState(vm)`，它的作用就是初始化组件状态（props，computed，data）：

```js
export function initState(vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe((vm._data = {}), true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

别的不看，只看这个：

```js
if (opts.props) initProps(vm, opts.props)
```

```js
function initProps(vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = (vm._props = {})
  // ...省略其他逻辑
}
```

这里的 propsData 就是数据源，他会从`vm.$options.propsData`上取，所我们传入的`propsData`经过`mergeOptions`后合并到`vm.$options`，再到这里进行 props 的初始化。

#### 绑定事件

可能有时候我们还想给组件绑定事件，其实这里应该很多小伙伴都知道怎么做，我们可以通过`vm.$on`给组件绑定事件，这个也是平时经常用到的一个 api

```js
const MessageBoxCtor = Vue.extend(MessageBox)
const messageBoxInstance = new MessageBox({
  propsData: {
    message: 'hello'
  }
}).$mount('#target')
messageBoxInstance.$on('some-event', () => {
  console.log('success')
})
```

#### 使用插槽

为了更加灵活的定制组件，我们还可以给组件传入插槽，比如组件可能是这样的：

```js
<template>
  <div class="message-box">
    {{ message }}
    <slot name="footer"/>
  </div>
</template>

<script>
  export default {
    props: {
      message: {
        type: String,
        default: ''
      }
    }
  }
</script>
```

这里我们先来分析下，如何才能给组件传入插槽内容？其实这里写的 template 会被 Vue 的编译器编译成一个 render 函数，组件渲染时执行的是这个渲染函数，我们先来看下这个 template 编译后的 render 是什么：

```js
function render() {
  with (this) {
    return _c(
      'div',
      {
        staticClass: 'message-box'
      },
      [_v(_s(message)), _t('footer')],
      2
    )
  }
}
```

这里的`_t('footer')`就是渲染插槽时执行的函数，`_t`是`renderSlot`的缩写，你可以在源码目录的`src/core/instance/render-helpers/render-slot.js`中找到这个函数，为方便理解，我将这个函数做了些简化，去除掉了不重要的逻辑：

```js
export function renderSlot(name, fallback, props) {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes /** Array<VNode> */
  if (scopedSlotFn) {
    // scoped slot
    props = props || {}
    nodes = scopedSlotFn(props) || fallback
  } else {
    nodes = this.$slots[name] || fallback
  }
  return nodes
}
```

这个函数就是从`$scopedSlots`中取到对应的插槽函数，然后执行这个函数，得到虚拟节点，然后返回虚拟节点，需要注意的是，Vue 在`2.6.x`版本中已经将普通插槽和作用域插槽都整合在了`$scopedSlots`，所有的插槽都是返回虚拟节点的函数，`renderSlot`里面的`else`分支中从`$slots`取插槽是兼容以前的写法的，所以说如果你用的是`Vue2.6.x`版本的话，你是不需要去关心`$slots`的。

由于`renderSlot`执行在组件实例的作用域中，所以`this.$scopedSlots`这里的`this`是组件的实例`vm`，所以我们只需要在创建完组件实例后，在实例上添加`$scopedSlots`就可以了，再根据之前的分析，这个`$scopedSlots`是一个对象，其中的 key 是插槽名称，value 是一个返回虚拟节点数组的函数：

```js
const MessageBoxCtor = Vue.extend(MessageBox)
const messageBoxInstance = new MessageBox({
  propsData: {
    message: 'hello'
  }
})
const h = this.$createElement
messageBoxInstance.$scopedSlots = {
  footer: function() {
    return [h('div', 'slot-content')]
  }
}
messageBoxInstance.$mount('#target')
```

这里需要注意的是`$mount`一定要在设置完`$scopedSlots`之后，因为`$mount`中会执行渲染函数，我们要保证在执行渲染函数时能获取到`$scopedSlots`。

如果你想使用作用域插槽，也很简单，和普通插槽是一样的，只需要在函数中接收参数就可以了：

```html
<slot name="head" :message="message"></slot>
```

```js
messageBoxInstance.$scopedSlots = {
  footer: function(slotData) {
    return [h('div', slotData.message)]
  }
}
```

这样就可以成功渲染出`message`了。

### 总结

本文章针对`Vue.extend`这个 API，从源码到使用彻底进行了分析，相信你看过后应该能游刃有余地使用了。当然只有这些理论肯定还是不够的，我们需要在实际开发场景中找到它的应用场景，比如我们常用的命令式弹窗组件，在很多 UI 组件库里都能看到它，你也可以去看一下 ElementUI 的 Message 组件的源码，看看是怎么实现的，有了这篇文章的基础后相信你一定能看懂。
