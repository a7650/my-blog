经常见到有人问看某某某源码有没有用，从我个人的经历来说（虽然我的经历也不长），我觉得是很有用的，而且非常有用。看一些框架和库的源码可以让我们了解到其中的某些特性是怎么实现的，使我们对这些技术更加熟悉；另一方面，看源码的过程也是个学习的过程，你可以学习整个项目的架构，学习作者的思路，学习某个函数的实现，或者代码风格等等，因为很多东西是自己无论如何也想不到的，所以我们可以从一些优秀的项目的源码中去学习、去借鉴，然后应用到自己的代码里，这样就会潜移默化的提升自己的能力，我觉得这比知道某些功能是怎么实现的要更加重要。

其实我也一直想写一些 Vue 源码分析的文章，但是现在网上分析 Vue 源码的文章随便都能找出来百八十篇，实在是太多了，我也不想再重复去写那么多。所以我想了个办法，找一些面试题，从源码的角度来分析，这样既能多看几道面试题，又能巩固源码的学习，岂不是一举两得，所以我打算每周找几个 Vue 面试题来分析下，希望各位小伙伴们看完能有一点收获。

由于通过面试题分析的话并不会从头去看源码，而且我也不会写的特别细致，所以本文章适合有基础的同学看，否则某些地方可能看不太明白。我这些分析只是辅助，还是建议大家有时间的话能完整的看一看源码，毕竟多看优秀的项目才能提升自己的代码能力，那废话不多说，我们开始看题吧。

#### 在使用计算属性时，函数名和 data 数据源中的数据可以同名吗？

**答案**： 不可以重名，不仅仅是计算属性和 data，其他的如 props，methods 都不可以重名，因为 Vue 会把这些属性挂载在组件实例上，直接使用 this 就可以访问，如果重名就会导致冲突。

**源码分析**：
在组件初始化的时候会执行\_init 函数

```js
Vue.prototype._init = function (options?: Object) {
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
      console.log('_isComponent',options)
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
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
}
```

这里面执行了一个 initState 的方法，这个方法就是初始化数据的关键方法

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

可以看到这里初始化了很多数据，有 props，methods，data，computed，watch 这几个。对于上面这个问题，我们只看 initComputed 这个方法

```js
function initComputed(vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = (vm._computedWatchers = Object.create(null))
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(`Getter is missing for computed property "${key}".`, vm)
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

在这个方法的最底部，做了一个判断，回去查你定义的 computed 的 key 值在 data 和 props 中有没有存在，这个时候 props 和 data 都已经初始化完成了，且已经挂载到了组件实例上，你的 computed 如果有冲突的话就会报错了，其实这几个初始化数据的方法内部都有做这些检测。

#### 怎么给 vue 定义全局的方法？

**答案**：目前一般有两种做法：一是给直接给 Vue.prototype 上添加，另外一个是通过 Vue.mixin 注册。但是为什么 prototype 和 mixin 里面的方法可以在组件访问到呢？

**源码**：由于这里牵扯的比较多，我只说关键点，可能不是很详细，希望各位小伙伴们有时间自己去看一下。
我们都知道 Vue 内部很多操作都是通过虚拟节点进行的，在初始化时候会执行创建虚拟节点的操作，这个就是通过一个叫 createElement 的函数进行的，就是渲染函数的第一个参数，在 createElement 内如果发现一个节点是组件的话，会执行 createComponent 函数

```js
export function createComponent(
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  //省略其他逻辑...
  // ......
  const baseCtor = context.$options._base
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // ......
}
```

这里是创建了一个组件的构造函数，baseCtor 是 Vue 的构造函数，其实下面执行的就是 Vue.extend()方法，这个方法是 Vue 构造函数上的一个静态方法，应该有不少小伙伴都用过这个方法，我们来看下这个方法做了什么事情

```js
/**
 * Class inheritance
 */
Vue.extend = function(extendOptions: Object): Function {
  //省略其他逻辑...
  // ......
  const Sub = function VueComponent(options) {
    this._init(options)
  }
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  Sub.cid = cid++
  Sub.options = mergeOptions(Super.options, extendOptions)
  // ......
}
```

创建了一个 Sub 函数，并且继承了将 prototype 指向了`Object.create(Super.prototype)`，是 js 里一个典型的继承方法，要知道，最终组件的实例化是通过这个 Sub 构造函数进行的，在组件实例内访问一个属性的时候，如果本实例上没有的话，会通过原型链向上去查找，这样我们就可以在组件内部访问到 Vue 的原型。

那么 mixin 是怎么实现的呢？其实上面这段代码还有一个是 mergeOptions 的操作，这个 mergeOptions 函数做的事情是将两个 options 合并在一起，这里我就不展开说了，因为里面的东西比较多。这里其实就是把 Super.options 和我们传入的 options 合并在一起，这个 Super 的 options 其实也就是 Vue 的 options，在我们使用 Vue.mixin 这个方法的时候，会把我们传入的 options 添加到 Vue.options 上

```js
export function initMixin(Vue: GlobalAPI) {
  Vue.mixin = function(mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```

这样 Vue.options 上就会有我们添加到属性了，在 extend 的时候这个属性也会扩展到组件构造函数的 options 上，

然后在组件初始化的时候，会执行 init 方法：

```js
Vue.prototype._init = function(options?: Object) {
  //省略其他逻辑
  // ......
  // merge options
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  // ......
}
```

里面判断到是组件时会执行 initInternalComponent 这个方法

```js
export function initInternalComponent(
  vm: Component,
  options: InternalComponentOptions
) {
  const opts = (vm.$options = Object.create(vm.constructor.options))
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent // 父Vnode,activeInstance
  opts._parentVnode = parentVnode // 占位符Vnode
  opts._parentElm = options._parentElm
  opts._refElm = options._refElm

  const vnodeComponentOptions = parentVnode.componentOptions // componentOptions是createComponent时候传入new Vnode()的
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

这里又通过`const opts = vm.$options = Object.create(vm.constructor.options)`将组件构造函数的 options 赋值给了 vm.\$options，这里的`vm.constructor.options`就是刚才和 Vue.options 合并后的组件构造函数上的 options，这样我们就在组件内部拿到了 Vue.mixin 定义的方法。

#### Vue 中怎么重置 data

**答案**：使用 `Object.assign(this.$data, this.$options.data())`即可。很多人都用过这个方法，在网上查到的也都是这个方法，那么这个方法的原理是什么，为什么这样写能达到效果呢？

**源码**：
这个方法就是一个简单的对象合并的方法，我们都知道`this.$options.data()`就是在组件内部书写的 data 函数，执行这个函数就会返回一份初始的 data 数据，那这个\$data 是个什么呢，它是在什么时候定义的呢？其实第一题里面也说过在初始化的时候执行了一个叫做 initState 的方法，里面又执行了 initData 来初始化数据：

```js
function initData(vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function' ? getData(data, vm) : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' &&
      warn(
        'data functions should return an object:\n' +
          'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
        vm
      )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(`Method "${key}" has already been defined as a data property.`, vm)
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' &&
        warn(
          `The data property "${key}" is already declared as a prop. ` +
            `Use prop default value instead.`,
          vm
        )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

我们可以看到这个方法在最开始执行了 data 函数，然后又把返回值赋值给了 vm.\_data，在函数的最后又执行了`proxy(vm,_data, key)`，我们来看下 proxy 方法：

```js
export function proxy(target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter() {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter(val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

这个方法就是通过`Object.defineProperty`执行了一层代理，这样我们在组件内部访问一个属性时，比如 this.name 其实访问的是 this.\_data.name。到这肯定会有小伙伴有疑问：上面那个答案是\$data，而这里是\_data，这是怎么回事呢？

其实在 Vue 这个构造函数初始化的时候还执行了一个方法

```js
function Vue(options) {
  if (process.env.NODE_ENV !== 'production' && !(this instanceof Vue)) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

这里是 Vue 构造函数初始化的过程，我们看下这个 stateMixin 方法，

```js
export function stateMixin(Vue: Class<Component>) {
  // flow somehow has problems with directly declared definition object
  // when using Object.defineProperty, so we have to procedurally build up
  // the object here.
  const dataDef = {}
  dataDef.get = function() {
    return this._data
  }
  const propsDef = {}
  propsDef.get = function() {
    return this._props
  }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function(newData: Object) {
      warn(
        'Avoid replacing instance root $data. ' +
          'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function() {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)
  // 省略其他逻辑...
  //......
}
```

这里定义了一个 dataDef，dataDef 的 get 方法返回了\_data，又通过`Object.defineProperty`将\$data指向了dataDef，这样我们访问 \$data 的时候其实访问的是\_data，而\_data 里保存的就是最终的 data 数据，所以我们才可以使用`Object.assign(this.$data, this.$options.data())`来达到重置数据的目的。

