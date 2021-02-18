> 由于通过面试题分析的话并不会从头去看源码，而且我也不会写的特别细致，所以本文章适合有基础的同学看，否则某些地方可能看不太明白。我这些分析只是辅助，还是建议大家有时间的话能完整的看一看源码，毕竟多看优秀的项目才能提升自己的代码能力。

#### 如何在子组件中访问父组件的实例？

**答案**：通过\$parent 就可以访问到父组件的实例了，除了\$parent，我们还可以通过\$children 访问子组件的实例。相信这个答案各位小伙伴都知道，但是这个\$parent 和\$children 是通过什么方式实现的呢，或者说，Vue 内部是如何建立这种父子组件关系的？

**源码解析**：
如果我们写一个组件 A，初始化组件的时候都会执行 A.\$mount 方法，将组件进行挂载，\$mount 方法实际上会执行 mountComponent

```js
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  //...省略其他逻辑
  // 创建一个组件的渲染Watcher
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
  //...
  new Watcher(vm, updateComponent, noop, null, true /* isRenderWatcher */)
  //...省略其他逻辑
}
```

这个函数最关键的地方是创建了一个渲染 Watcher，其内部是执行了 vm.\_update(vm.\_render(), hydrating)，\_render 主要就是返回一个 vnode，就是虚拟节点，然后\_update 通过这个 vnode 去 patch 组件，那么我们看一下\_update 做了什么事情

```js
export let activeInstance: any = null
//...
Vue.prototype._update = function(vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  if (vm._isMounted) {
    callHook(vm, 'beforeUpdate')
  }
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  //...省略其他逻辑
  vm.$el = vm.__patch__(
    vm.$el,
    vnode,
    hydrating,
    false /* removeOnly */,
    vm.$options._parentElm,
    vm.$options._refElm
  )
  //...省略其他逻辑
  activeInstance = prevActiveInstance
}
```

这个函数外部定义了一个 activeInstance 变量，在执行\_update 的时候通过`const prevActiveInstance = activeInstance`将 activeInstance 保存了起来，又通过`activeInstance = vm`把当前组件实例赋值给了 activeInstance，这个 activeInstance 是在函数外部定义的一个变量，其他文件也可以 import 这个变量，activeInstance 的作用稍后会讲到。我们先看下 patch，patch 过程比较复杂，我只讲和本题相关的一些逻辑，假设我们的组件 A 有一个子组件 A1，在 patch 的时候，将会执行到 A1 组件的 hook.init 函数

```js
init(
    vnode: VNodeWithData,
    hydrating: boolean,
    parentElm: ?Node,
    refElm: ?Node
  ): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance,
        parentElm,
        refElm
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
```

这里执行了 createComponentInstanceForVnode 这个函数，注意传入的第二个参数是 activeInstance，这个 activeInstance 是从 mountComponent 那引入的，在 mountComponent 的时候已经把这个变量设置成了 A，_注意由于这个 init 函数初始化的是 A1 组件，那么对于 A1 组件来说，activeInstance 就是它的父组件_。看下 createComponentInstanceForVnode 函数

```js
export function createComponentInstanceForVnode(
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
  parentElm?: ?Node,
  refElm?: ?Node
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    parent,
    _parentVnode: vnode,
    _parentElm: parentElm || null,
    _refElm: refElm || null
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```

刚才那个 activeInstance 就是这个 parent 参数，然后又放在了 options 中，执行了 new vnode.componentOptions.Ctor(options)函数，这个 vnode.componentOptions.Ctor 是组件的构造函数，是在创建 vnode 时通过 Vue.extend 创建的，关于 Vue.extend 做的事情，可查看我上一篇文章，这个 Ctor 执行了这一段函数

```js
function VueComponent(options) {
  this._init(options)
}
```

这里又回到了\_init

```js
Vue.prototype._init = function(options?: Object) {
  //...省略其他逻辑
  if (options && options._isComponent) {
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  //...省略其他逻辑
  initLifecycle(vm)
  //...省略其他逻辑
}
```

组件会执行到 initInternalComponent 函数

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

这里面有一段逻辑是 opts.parent = options.parent，就是把 options.parent 这个属性赋值给了 vm.\$options，这里 options.parent 就是 createComponentInstanceForVnode 中的 parent，也就是 activeInstance，那么现在 vm.\$options.parent 就是 activeInstance。
执行完这些后再回到\_init 中，我们看到下面还有一段函数 initLifecycle(vm)

```js
export function initLifecycle(vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```

这里拿到了 vm.\$options.parent，通过刚才的 initInternalComponent 我们可以知道，这个 parent 其实就是 activeInstance，然后又通过 parent.\$children.push(vm)往 parent 的\$children 中添加自己，又通过 vm.\$parent = parent 将 parent 赋值给\$parent，所以这个时候，A1 的\$parent 就是 A，A 的\$children 里面也会有 A1，执行完这些后，再回到最开始的 init 方法

```js
init(
    vnode: VNodeWithData,
    hydrating: boolean,
    parentElm: ?Node,
    refElm: ?Node
  ): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance,
        parentElm,
        refElm
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
```

这个 init 方法又执行了\$mount 方法，接着又是 mountComponent > \_update

```js
Vue.prototype._update = function(vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  if (vm._isMounted) {
    callHook(vm, 'beforeUpdate')
  }
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  //...省略其他逻辑
  vm.$el = vm.__patch__(
    vm.$el,
    vnode,
    hydrating,
    false /* removeOnly */,
    vm.$options._parentElm,
    vm.$options._refElm
  )
  //...省略其他逻辑
  activeInstance = prevActiveInstance
}
```

注意这里仍然是在 A 的子组件 A1 中，\_update 会执行 A1 的 patch，如果 A1 有子组件的话，就又会重复上面的操作，整个过程其实是一个递归的过程，每次递归都会对 activeInstance 赋值，最终在 patch 完成之后，会通过`activeInstance = prevActiveInstance`将 activeInstance 恢复，这样就可以确保 patch 过程中的 activeInstance 是父组件实例，patch 完成之后也不会影响其他逻辑，这样 Vue 就建立了层层的父子级关系。

#### vue 怎么实现强制刷新组件？

**答案**：Vue 提供了一个 API：\$forceUpdate，可以强制渲染组件。

**源码解析**：Vue 内部会监听组件 data，并自动收集依赖，在属性发生变化时自动通知更新，触发组件的重新渲染。一般情况下，我们是不需要关心这个过程的，但有时我们想强制刷新视图，就要使用\$forceUpdate 这个函数了，那这个函数做了什么事情呢，我们先来看下组件挂载过程的 mountComponent 函数

```js
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  //省略其他逻辑...
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
  //省略其他逻辑...
  new Watcher(vm, updateComponent, noop, null, true /* isRenderWatcher */)
  //省略其他逻辑...
}
```

我们只看和本题相关的逻辑，这里创建了一个 updateComponent 函数，然后又将 updateComponent 参数实例化了一个 Watcher，这个 updateComponent 函数的细节我就不展开讲了，总之我们在更新组件属性的后，Vue 内部也会执行到这个函数，使视图重新渲染。还有这个 Watcher 类比较多，我就不贴出来占地方了，与本题相关的关键的一个地方是下面这个

```js
// class Watcher
if (isRenderWatcher) {
  vm._watcher = this
}
```

将组件实例的\_watcher 属性指向了 watcher 本身。这样我们就可以通过 vm.\_watcher 访问到 watcher 了。除了上面这些操作之外，Vue 原型上还定义了\$forceUpdate 方法

```js
Vue.prototype.$forceUpdate = function() {
  const vm: Component = this
  if (vm._watcher) {
    vm._watcher.update()
  }
}
```

可以看到这个 forceUpdate 其实就是执行了 watcher 的 update，watcher.update()最终会执行到 updateComponent 函数，从而触发一次更新。

可以看到，\$forceUpdate 其实就是主动触发一次更新操作，达到强制更新的目的。
