Vue3 和 Vue2 的响应式有很大的不同，由于 Vue3 使用 Proxy 代替了 defineProperty，使得 Vue3 比 Vue2 在响应式数据处理方面有着更好的性能，更简洁高效的处理方式，还实现了诸多在 Vue2 上无法实现的功能。此外 Vue3 的响应式库 reactivity 是一个单独的包，它可以不依赖 Vue 运行，意味着我们可以将它运行在其他框架里。事实上，Vue3 的响应式库的实现方式以及市面上其他的大多数响应式库（如 observer-util，meteor 等）的实现方式都是类似的，Vue 也是参考这些库实现的，所以我们还是很有必要去研究一下的，毕竟咱也不能落伍了 😄，那么各位小伙伴们下面就跟我一起来看下这个 `@vue/reactivity` 究竟是怎么实现的。

> 本文章的源码已经发在了我的 git 上，可以前往查看：[reactivity](https://github.com/a7650/reactivity)

#### 阅读本文章之前你要先了解以下知识点

- [WeakMap](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)
- [Set](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Set)
- [Reflect](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect)
- [Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

上面这些有不了解的同学可以直接点链接查看详细的文档，文章里面就不再解释了。

--

我们首先看一个使用 reactivity 的例子

```ts
// 创建一个响应式对象
const state = reactive({ count: 1 })

// 执行effect
effect(() => {
  console.log(state.count)
})

state.count = 2 // count改变时执行了effect内的函数，控制台输出2
```

这个例子通过 reactive 创建了一个响应式对象 state，然后调用 effect 执行函数，这个函数内部访问了 state 的属性，随后我们更改这个 state 的属性，这时，effect 内的函数会再次执行。

这样一个响应式数据的通常实现的方式是这样的

1. 定义一个数据为响应式（通常通过 defineProperty 或者 Proxy 拦截 get、set 等操作）
2. 定义一个副作用函数（effect），这个副作用函数内部访问到响应式数据时会触发 1 中的 getter，进而可以在这里将 effect 收集起来
3. 修改响应式数据时，就会触发 1 中的 setter，进而执行 2 中收集到的 effect 函数

> **关于 effect**： effect 在 Vue 里通常叫做副作用函数，因为这种函数内通常执行组件渲染，计算属性等其他任务。在其他库里面可能叫观察者函数（observe）或其他，个人能理解到是什么意思就好，由于本篇文章是分析 Vue3 的，所以统一叫副作用函数（effect）

根据以上的思路，我们就可以开始动手实现了

#### reactive

首先我们需要有一个 reactive 函数来将我们的数据变为响应式。

```ts
// reactive.ts
import { baseHandlers } from './handlers'
import { isObject } from './utils'

type Target = object

const proxyMap = new WeakMap()

export function reactive<T extends object>(target: T): T {
  return createReactiveObject(target)
}

function createReactiveObject(target: Target) {
  // 只对对象添加reactive
  if (!isObject(target)) {
    return target
  }
  // 不能重复定义响应式数据
  if (proxyMap.has(target)) {
    return proxyMap.get(target)
  }
  // 通过Proxy拦截对数据的操作
  const proxy = new Proxy(target, baseHandlers)
  // 数据添加进ProxyMap中
  proxyMap.set(target, proxy)
  return proxy
}
```

这里主要对数据做了简单的判断，关键是在`const proxy = new Proxy(target, baseHandlers)`中，通过 Proxy 对数据进行处理，这里的`baseHandlers`就是对数据的 get，set 等拦截操作，下面来实现下`baseHandlers`

#### get 收集依赖

首先实现下拦截 get 操作，使得访问数据的某一个 key 时，可以收集到访问这个 key 的函数（effect），并把这个函数储存起来。

```ts
// handlers.ts
import { track } from './effect'
import { reactive, Target } from './reactive'
import { isObject } from './utils'

export const baseHandlers: ProxyHandler<object> = {
  get(target: Target, key: string | symbol, receiver: object) {
    // 收集effect函数
    track(target, key)
    // 获取返回值
    const res = Reflect.get(target, key, receiver)
    // 如果是对象，要再次执行reactive并返回
    if (isObject(res)) {
      return reactive(res)
    }
    return res
  }
}
```

这里我们拦截到 get 操作后，通过 track 收集依赖，track 函数做的事情就是把当前的 effect 函数收集起来，执行完 track 后，再获取到 target 的 key 的值并返回，注意这里是判断了下 res 是否是对象，如果是对象的话要返回`reactive(res)`，是因为考虑到可能有多个嵌套对象的情况，而 Proxy 只能修改到到当前对象，并不能修改到子对象，所以在这里要处理下，下面我们需要再实现`track`函数

```ts
// effect.ts

// 存储依赖
type Deps = Set<ReactiveEffect>
// 通过key去获取依赖，key => Deps
type DepsMap = Map<any, Deps>
// 通过target去获取DepsMap，target => DepsMap
const targetMap = new WeakMap<any, DepsMap>()
// 当前正在执行的effect
let activeEffect: ReactiveEffect | undefined

// 收集依赖
export function track(target: object, key: unknown) {
  if (!activeEffect) {
    return
  }
  // 获取到这个target对应的depsMap
  let depsMap = targetMap.get(target)
  // depsMap不存在时新建一个
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  // 有了depsMap后，再根据key去获取这个key所对应的deps
  let deps = depsMap.get(key)
  // 也是不存在时就新建一个
  if (!deps) {
    depsMap.set(key, (deps = new Set()))
  }
  // 将activeEffect添加进deps
  if (!deps.has(activeEffect)) {
    deps.add(activeEffect)
  }
}
```

注意有两个 map 和一个 set，targetMap => depsMap => deps，这样就可以使我们通过 target 和 key 准确地获取到这个 key 所对应的 deps(effect)，把当前正在执行的 effect（activeEffect）存起来，这样在修改`target[key]`的时候，就又可以通过 target 和 key 拿到之前收集到的所有的依赖，并执行它们，这里有个问题就是这个`activeEffect`它是从哪里来的，get 是怎么知道当前正在执行的 effect 的？这个问题可以先放一放，我们后面再将，下面我们先实现这个 set。

#### 实现 set

```ts
// handlers.ts

export const baseHandlers: ProxyHandler<object> = {
  get() {
    //...
  },
  set(target: Target, key: string | symbol, value: any, receiver: object) {
    // 设置value
    const result = Reflect.set(target, key, value, receiver)
    // 通知更新
    trigger(target, key, value)
    return result
  }
}
```

我们在刚才的`baseHandlers`下面再加一个 set，这个 set 里面主要就是赋值然后通知更新，通知更新通过`trigger`进行，我们需要拿到在 get 中收集到的依赖，并执行，下面来实现下 trigger 函数

```ts
// effect.ts

// 通知更新
export function trigger(target: object, key: any, newValue?: any) {
  // 获取该对象的depsMap
  const depsMap = targetMap.get(target)
  // 获取不到时说明没有触发过getter
  if (!depsMap) {
    return
  }
  // 然后根据key获取deps，也就是之前存的effect函数
  const effects = depsMap.get(key)
  // 执行所有的effect函数
  if (effects) {
    effects.forEach((effect) => {
      effect()
    })
  }
}
```

这个 trigger 就是获取到之前收集的 effect 然后执行。

其实除了 get 和 set，还有个常用的操作，就是删除属性，现在我们还不能拦截到删除操作，下面我们来实现下

#### 实现 deleteProperty

```ts
export const baseHandlers: ProxyHandler<object> = {
  get() {
    //...
  },
  set() {
    //...
  },
  deleteProperty(target: Target, key: string | symbol) {
    // 判断要删除的key是否存在
    const hadKey = hasOwn(target, key)
    // 执行删除操作
    const result = Reflect.deleteProperty(target, key)
    // 只在存在key并且删除成功时再通知更新
    if (hadKey && result) {
      trigger(target, key, undefined)
    }
    return result
  }
}
```

我们在刚才的`baseHandlers`里面再加一个`deleteProperty`，它可以拦截到对数据的删除操作，在这里我们需要先判断下删除的 key 是否存在，因为可能用户会删除一个并不存在 key，然后执行删除，我们只在存在 key 并且删除成功时再通知更新，因为如果 key 不存在时，这个删除是无意义的，也就不需要更新，再有就是如果删除操作失败的话，也不需要更新，最后直接触发`trigger`就可以了，注意这里的第三个参数即 value 是`undefined`

现在我们已经实现了`get`，`set`，`deleteProperty`这三种操作的拦截，还记不记得在`track`函数中的`activeEffect`，那里留了个问题，就是这个`activeEffect`是怎么来的？，在最开始的例子里面，我们要通过 effect 执行函数，这个`activeEffect`就是在这里设置的，下面我们来实现下这个`effect`函数。

```ts
// effect.ts

type ReactiveEffect<T = any> = () => T
// 存储effect的调用栈
const effectStack: ReactiveEffect[] = []

export function effect<T = any>(fn: () => T): ReactiveEffect<T> {
  // 创建一个effect函数
  const effect = createReactiveEffect(fn)
  return effect
}

function createReactiveEffect<T = any>(fn: () => T): ReactiveEffect<T> {
  const effect = function reactiveEffect() {
    // 当前effectStack调用栈不存在这个effect时再执行，避免死循环
    if (!effectStack.includes(effect)) {
      try {
        // 把当前的effectStack添加进effectStack
        effectStack.push(effect)
        // 设置当前的effect，这样Proxy中的getter就可以访问到了
        activeEffect = effect
        // 执行函数
        return fn()
      } finally {
        // 执行完后就将当前这个effect出栈
        effectStack.pop()
        // 把activeEffect恢复
        activeEffect = effectStack[effectStack.length - 1]
      }
    }
  } as ReactiveEffect<T>
  return effect
}
```

这里主要是通过`createReactiveEffect`创建一个 effect 函数，fn 就是调用 effect 时传入的函数，在执行这个 fn 之前，先通过`effectStack.push(effect)`把这个 effect 推入 effectStack 栈中，因为 effect 可能存在嵌套调用的情况，保存下来就可以获取到一个完整的 effect 调用栈，就可以通过上面的`effectStack.includes(effect)`判断是否存在循环调用的情况了，然后再`activeEffect = effect`设置 activeEffect，设置完之后再执行 fn，因为这个 activeEffect 是全局唯一的，所以我们执行 fn 的时候，如果内部访问了响应式数据，就可以在 getter 里拿到这个 activeEffect，进而收集它。

现在基本上是完成了，现在通过我们写的这个 reactivity 库就可以实现例子中的效果了，但是还有一些边界情况需要考虑，下篇文章就添加一些常见的边界情况处理。
