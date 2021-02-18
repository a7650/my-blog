### 前言

Vue 中的 computed 是一个日常开发中常用到的属性，也是面试中经常被问到的一个知识点，你几乎能在任何一个和 Vue 相关的面试题集锦里找到这样一个题目：methods 和 computed 有什么不同？你可能会毫不犹豫地回答："methods 不会被缓存，computed 会对计算结果进行缓存"。确实，这个缓存是一个主要的的特点，但是，这个缓存指的是什么？缓存是怎么实现的？哪种情况下不会被缓存？这个缓存什么时候会被重新求值？缓存有什么好处？除了缓存我们还可以问：怎样在计算属性中使用 setter？计算属性是否能依赖其他计算属性，内部的原理是什么？对于这些问题，可能很多人都不是很了解，不过没关系，这篇文章就带你来深入理解这个计算属性，任面试官怎么问都不怕。

> 本文使用的 Vue 源码版本是 2.6.11

### DEMO

我们先来看一个简单的例子，本文将会针对这个例子进行分析：

```html
<div id="app">
  <div @click="add">doubleCount：{{doubleCount}}</div>
</div>
<script>
  new Vue({
    el: '#app',
    name: 'root',
    data() {
      return {
        count: 1
      }
    },
    computed: {
      doubleCount() {
        return this.count * 2
      }
    },
    methods: {
      add() {
        this.count += 1
      }
    }
  })
</script>
```

这里使用了一个`doubleCount`计算属性，它的值是`count`的两倍，每次点击会使`count`的值加一，`doubleCount`也随之改变。

### 原理分析

> 首先你要对 Vue 的响应式系统原理有所了解，不了解的话可以先去网上搜一下这方面的文章。

> 本文贴的 Vue 源码并不是原版的源码，为了便于分析讲解，对原版的源码做了简化，去除了不重要的逻辑和边界情况的处理。

直接看源码

#### 初始化过程

组件初始化时会执行`initState`函数：

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

这里进行了 props，data 等属性的初始化，在初始化完 data 后执行`initComputed`进行计算属性的初始化，这也是为什么我们可以在计算属性中直接访问 props，data，methods，就是因为它的初始化发生在这三者之后，下面来看下`initComputed`的逻辑：

```js
// vm是组件实例,computed是我们在options中定义的对象。
function initComputed(vm: Component, computed: Object) {
  // 先创建一个watchers，是一个空对象
  const watchers = (vm._computedWatchers = Object.create(null))
  for (const key in computed) {
    // 获取这个计算属性的定义，对于刚才的例子，这个userDef就是doubleCount这个函数
    const userDef = computed[key]
    // 由于doubleCount是个函数，所以这里的getter还是doubleCount这个函数
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    // 创建一个Watcher，存入watchers中
    watchers[key] = new Watcher(
      vm,
      getter || noop,
      noop,
      computedWatcherOptions
    )
    // 待会讲
    defineComputed(vm, key, userDef)
  }
}
```

这个函数就是遍历定义的 computed，对每个计算属性都创建一个 Watcher，然后保存在 watchers 中，注意这个 watchers 是在 vm 的`_computedWatchers`属性上的，创建 watcher 的时候传入了一个`computedWatcherOptions`，是一个只有 lazy 属性的对象：

```js
const computedWatcherOptions = { lazy: true }
```

下面来简单看下 Watcher：

```js
class Watcher {
  constructor(
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    // options就是刚才的computedWatcherOptions，所以lazy为true
    this.lazy = !!options.lazy
    // 用来控制缓存，稍后讲
    this.dirty = this.lazy // true
    this.deps = [] // 收集的Dep
    // 求值方法，对于我们的例子而言，就是doubleCount函数
    this.getter = expOrFn
    // 初始化value，由于lazy为true，所以什么也不会执行，这里的value为undefined
    this.value = this.lazy ? undefined : this.get()
  }
}
```

这里关键的地方是`lazy`属性和`dirty`属性，`lazy`的作用为惰性求值，在初始化 value 时由于`lazy`为 true，所以并不会求值。`dirty`的作用我们稍后再说，接下来接着看 computed 的初始化。

在创建了 watcher 之后，还执行了`defineComputed(vm, key, userDef)`:

```js
export function defineComputed(
  target: any, // vm
  key: string, // computed的key：'doubleCount'
  userDef: Object | Function // 计算属性的值，doubleCount函数
) {
  // 使用defineProperty设置getter和setter
  Object.defineProperty(target, key, {
    enumerable: true,
    configurable: true,
    get: function computedGetter() {
      // 拿到initComputed中创建的Watcher
      const watcher = this._computedWatchers && this._computedWatchers[key]
      if (watcher) {
        // !!dirty为true时才会执行evaluate
        if (watcher.dirty) {
          // evaluate会对watcher进行求值，并将dirty置为false
          watcher.evaluate()
        }
        // 待会讲
        if (Dep.target) {
          watcher.depend()
        }
        // 返回watcher的值
        return watcher.value
      }
    }
  })
}
```

`defineComputed`主要是通过`defineProperty`设置了代理，通过实例访问计算属性时就会执行这个 get 函数。

#### 缓存的实现

设想初次渲染的场景，`count`值为 1，在模板中访问`doubleCount`属性，就会执行`defineComputed`中定义的 get 函数，这个 get 函数首先会拿到刚才在`initComputed`中定义的`watcher`，然后判断`watcher.dirty`，刚才创建的 watcher 的 dirty 为 true，所以会执行`watcher.evaluate()`，我们来看下这个`evaluate`方法

```js
class Watcher {
  constructor() {
    // ...
  }
  evaluate() {
    this.value = this.get() // get会对watcher求值，稍后细讲
    this.dirty = false // 重新将dirty置为false
  }
}
```

这里会执行 get 函数，get 函数会执行 watcher 的 getter，对于我们的例子而言就是执行`doubleCount`函数：`return this.count * 2`，由于初始的 count 为 1，所以这里会返回 2，然后将结果赋值给 value，再把 dirty 置为 false，这个时候 watcher 就有值了，再回到`defineComputed`的 get 中，最后执行`return watcher.value`返回了 watcher 的值，这样模板中就渲染出了`doubleCount：2`，下次我们再访问`doubleCount`的时候，比如在`mounted`中`console.log(this.doubleCount)`，就又会走到`defineComputed`的 get，这个时候由于`watcher.dirty`为`false`，所以就不会执行`watcher.evaluate()`了，也就不会执行`doubleCount`函数了，它将会直接返回`watcher.value`，也就是 2，这样就实现了缓存。

如果将`count`从 1 变成 2，那么我们下次访问`doubleCount`时，应该拿到 4 才对，那这个缓存是什么时候更新的，是怎么更新的呢？别急，我们接着来分析。

#### 缓存更新

首先我们先来回顾下 Vue 响应式系统的流程，Vue 的响应式系统主要是通过 Watcher、Dep 以及`Object.defineProperty`实现的，初始化 data 时，通过`Object.defineProperty`设置属性的 getter 和 setter，使属性变为响应式，然后在执行某些操作（渲染操作，计算属性，自定义 watcher 等）时，创建一个 watcher，这个 watcher 在执行求值操作之前会将一个全局变量`Dep.target`指向自身，然后在求值操作过程中如果访问了响应式属性，就会把当前的`Dep.target`也就是 watcher 添加到属性的 dep 中，然后在下次更新响应式属性时，就会从 dep 中找出收集的 watcher，然后执行`watcher.update`，执行更新操作。

> 概括的比较简略，如果你不明白的话，建议去网上搜一下这方面的文章

了解了响应式系统后，我们再来分析上文的初次渲染场景，在首次渲染时，访问`doubleCount`时执行了`watcher.evaluate()`函数，里面有一个求值操作`this.value = this.get()`，我们来看下`this.get`这个函数

```js
class Watcher {
  constructor() {
    // ...
  }
  get() {
    // targetStack保存了当前的watcher栈
    // 因为可能在watcher求值过程中又创建了其他watcher
    targetStack.push(this)
    // 将Dep.target指向自身
    Dep.target = this

    let value
    const vm = this.vm
    // 执行getter函数，对于我们的例子而言，getter就是doubleCount函数
    value = this.getter.call(vm, vm)

    // 当前watcher出栈
    targetStack.pop()
    // 恢复到上一个watcher
    Dep.target = targetStack[targetStack.length - 1]

    return value
  }
}
```

这里主要做的就是设置`Dep.target`，然后执行 getter，因为`doubleCount`函数中访问了`count`属性，所以会执行到`count`的 getter 中：

```js
function defineReactive(obk, key, val) {
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      const value = val
      // 刚才定义的Dep.target，也就是计算属性的watcher
      if (Dep.target) {
        // 执行depend收集依赖
        dep.depend()
      }
      return value
    },
    set: function reactiveSetter(newVal) {
      // ...稍后讲
    }
  })
}
```

这个 get 主要就是执行`dep.depend()`收集依赖：

```js
class Dep {
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
}
```

这里执行了 watcher 的 addDep，将自身作为参数传入：

```js
class Watcher {
  addDep(dep) {
    dep.addSub(this)
    this.deps.push(dep)
  }
}
```

将 dep 添加到 watcher 自身的 deps 中，这里执行了`dep.addSub(this)`，参数是自身，又回到了 dep：

```js
class Dep {
  addSub(sub) {
    this.subs.push(sub)
  }
}
```

这个函数就是把 watcher 添加到自身的 subs 中，看似很绕，其实很好理解，就是分别去 dep 和 watcher 中将对方添加到自身的某个属性中，这样执行完之后，`dep.subs`中会是`[计算属性watcher]`，而`watcher.deps`会是`[count的dep]`，两者中都有对方的引用，这里可以得出一个结论就是 **调用某一个 dep 的 depend 方法时，会把 Dep.target 添加到自身的 subs 中（稍后会用到）** ，这是在初始化取值时做的操作，当设置了`count`为 2 时，就会走到 count 的 setter 逻辑中：

```js
function defineReactive(obk, key, val) {
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      //...
    },
    set: function reactiveSetter(newVal) {
      val = newVal
      const subs = dep.subs.slice()
      // 遍历subs，执行update函数
      for (let i = 0, l = subs.length; i < l; i++) {
        subs[i].update()
      }
    }
  })
}
```

这里将之前存的 watcher 取出，遍历并执行`watcher.update`：

```js
class Watcher {
  update() {
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
}
```

这里是关键的逻辑，由于计算属性的 lazy 为 true，所以这里会执行`this.dirty = true`的逻辑，到这里就完了。这里可能有小伙伴会很疑惑：这个逻辑如果到这里就完了，那么计算属性在哪里重新求值呢？视图在哪里重新渲染呢？如果照着这个逻辑的话，计算属性根本不会更新，视图也会不会重新渲染，那么问题出在哪里呢？

#### 视图如何更新

其实我们一直忽略了一个东西，那就是渲染 watcher，在渲染时是先执行渲染 watcher 的，然后渲染 watcher 中执行渲染函数，这时候在渲染函数会访问到`doubleCount`，然后执行`defineComputed`中定义的 getter，getter 中又执行了我们刚才说的`watcher.evaluate()`、`watcher.get()`等逻辑，那么我们再来分析下这个`watcher.get()`：

```js
class Watcher {
  constructor() {
    // ...
  }
  get() {
    // doubleCount的访问是发生在渲染watcher中的
    // 所以在执行下面这行代码之前，targetStack里面是：[渲染watcher]
    targetStack.push(this) // 执行这段代码后，targetStack里面是：[渲染watcher，计算watcher]
    Dep.target = this

    let value
    const vm = this.vm
    // 还是之前的逻辑，收集依赖
    // count的dep.subs中会是[计算属性watcher]，计算watcher的deps会是[count的dep]
    value = this.getter.call(vm, vm)

    // 当前watcher出栈
    targetStack.pop() // 执行完这段代码后，targetStack里面是：[渲染watcher]
    // 恢复到上一个watcher
    Dep.target = targetStack[targetStack.length - 1] // Dep.target是：渲染watcher

    return value
  }
}
```

执行完这个 get 后返回到`defineComputed`的 getter 中：

```js
get: function computedGetter() {
  const watcher = this._computedWatchers && this._computedWatchers[key]
  if (watcher) {
    if (watcher.dirty) {
      // 对watcher求值
      watcher.evaluate()
    }
    // watcher执行完求值后，Dep.target是渲染watcher，所以这里是有值的
    if (Dep.target) {
      // 执行watcher的收集依赖操作
      watcher.depend()
    }
    return watcher.value
  }
}
```

由于`Dep.target`有值，所以会执行`watcher.depend()`，来看下这个 depend：

```js
class Watcher {
  constructor() {
    // ...
  }
  depend() {
    // 上文已经分析过，计算watcher的deps是：[count的dep]
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
}
```

这里遍历 deps，并执行 dep 的 depend 方法，还记得这个方法和那个结论吗？**调用某一个 dep 的 depend 方法时，会把 Dep.target 添加到自身的 subs 中**，上文已经分析过 count 的 dep.subs 中是`[计算属性watcher]`，此时的`Dep.target`是渲染 watcher，那执行完这个 depend 后，count 的 dep.subs 中就是`[计算属性watcher, 渲染watcher]`。

到这里可能大家就明白了，更新响应式属性时，在 count 的 setter 中，遍历了 dep 的 subs 并执行 update 方法，这时候的 subs 里不只有计算属性的 watcher，还有渲染 watcher，我们再来看 update 方法：

```js
class Watcher {
  update() {
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
}
```

会先执行计算属性的 update，将`dirty`置为 true，然后执行渲染 watcher 的 update，渲染 watcher 的`lazy`和`sync`都为 false，所以会执行`queueWatcher(this)`，这个`queueWatcher`方法你可以不用关心它的作用，其实最终它会执行渲染 watcher 中的渲染函数，那在执行渲染函数时，又访问到了`doubleCount`：

```js
get: function computedGetter() {
  const watcher = this._computedWatchers && this._computedWatchers[key]
  if (watcher) {
    // 这个时候dirty已经是true了，表示它需要更新
    if (watcher.dirty) {
      // 对watcher求值，执行doubleCount函数
      // 执行完之后，watcher.value就会从2变为4
      watcher.evaluate()
    }
    if (Dep.target) {
      watcher.depend()
    }
    return watcher.value // 返回4
  }
}
```

由于我们已经在 update 阶段把 dirty 变为 true 了，所以此时会执行`watcher.evaluate()`，这样`doubleCount`就更新了，就会在页面上渲染出 4 了，如果我们再修改 count 的值，就会重新执行上文的逻辑。

如果不在模板中使用`doubleCount`，只通过 watch 监听计算属性，也是相似的逻辑，只不过是把渲染 watcher 换成 user watcher，你也可以自行打个断点分析下整个流程。

另外如果是多层嵌套计算属性的情况，可能比较复杂，不过思路还是上文的思路，最终 count 的 dep.subs 就是类似于这样的：`[AAA计算watcher,AA计算watcher，A计算watcher，渲染watcher]`

### 总结

通过本文可以总结出以下两点：

1. 计算属性watcher的lazy为true，当修改响应式属性执行`watcher.update`时，并不会对watcher求值，而是将`watcher.dirty`置为true，当下次访问这个计算属性时，发现dirty为true，这时候才会对watcher求值。
2. 如果计算属性的依赖没有发生改变，那么无论我们访问多少次都不会重新求值，会直接从`watcher.value`返回我们需要的值。

很多Vue性能优化的文章里都会提到：将一些需要进行大量计算的操作或者需要频繁执行的操作放在计算属性里。其利用的就是计算属性缓存的特点，减少无意义的计算。

除了本文讲的内容，计算属性还支持自定义setter，以及传入其他option，不过比较简单，你可以自行看源码分析，如果你彻底理解了本文内容，那么以后无论是面试还是日常开发，相信你定能游刃有余。