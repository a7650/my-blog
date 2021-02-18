> 本文章接上一篇

在上一篇文章里已基本实现了一个简单的 reactivity 系统，但是还有一些边界情况需要处理，我们来看一下下面这个例子

```ts
const state = reactive({ counts: { a: 1, b: 2 } })

effect(() => {
  console.log(Object.keys(state.counts))
})

Reflect.set(state.counts, 'c', 3) //并没有触发effect
```

按照之前的设想，在给 state.counts 添加了`c`属性后，应该会执行 effect，并输出`['a', 'b', 'c']`，但是事实上并没有触发，其实这里的原因是我们并没有拦截到`Object.keys`这个操作，如果你有仔细看过 Proxy 的文档的话，你应该能发现在 handlers 里面还有一个属性`ownKeys`，它可以帮我们拦截`Object.keys()`,`Reflect.ownKeys()`,`Object.getOwnPropertyNames()`,`Object.getOwnPropertySymbols()`这些操作，知道这些后，我们就可以继续进行扩展了

#### ownKeys

首先添加两个枚举，一会将用到

```ts
// operations.ts

//收集依赖的操作类型
export const enum TrackOpTypes {
  GET = 'get',
  ITERATE = 'iterate'
}

//触发依赖的操作类型
export const enum TriggerOpTypes {
  SET = 'set',
  ADD = 'add',
  DELETE = 'delete'
}
```

然后在之前的`baseHandlers`里添加`ownKeys`

```ts
export const ITERATE_KEY = Symbol()

export const baseHandlers: ProxyHandler<object> = {
  get() {
    //...
  },
  set() {
    //...
  },
  deleteProperty() {
    //...
  },
  ownKeys(target: Target): (string | number | symbol)[] {
    track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
    return Reflect.ownKeys(target)
  }
}
```

在这个`ownKeys`里面直接执行 track 收集依赖，注意我们之前的 track 只有两个参数`(target, key)`，我们等下还要对 track 做下修改，使它接收参数是`(target, type, key)`，`type`表示收集依赖的类型，就是刚才定义的`TrackOpTypes`，现在我们有`GET`和`ITERATE`，我们在 ownKeys 里就用`ITERATE`表明它是一个迭代操作，然后第三个参数`ITERATE_KEY`触发收集的 key 值，因为`Object.keys()`这种操作不是针对具体的某一个 key 的，所以我们就用`ITERATE_KEY`，其中`ITERATE_KEY = Symbol()`。

然后把 track 的参数改下

```ts
function track(target: object, type: TrackOpTypes, key: unknown) {
  //...
}
```

其实这个 type 我们现在也用不上，不过为了和一会的的 trigger 统一，还是加上的好，方便以后其他功能使用。

好了，写完 ownKeys 后，按照之前的逻辑，其实就可以收集到依赖了，因为传的 key 是`ITERATE_KEY`，所以我们获取 ownKeys 的依赖时，也要从`ownKeys`取。那么我们要在什么时候把这些依赖取出来呢，其实很简单，因为 ownKeys 针对的迭代操作是不关心属性的修改的，只关心属性的添加或删除，换句话说我们把`{ a: 1, b: 2 }`改成`{ a: 1, b: 3 }`，`Object.keys()`拿到的结果是不会变的，都是`['a','b']`，那有人可能会问了：和`Object.keys()`对应的还有`Object.values()`，这个总会变吧？其实，`Object.values()`内部还是会触发`ownKeys`，它是先拿的 key，再用 key 去拿 value，这个过程中不但触发了`ownKeys`，还触发了每个属性的`get`，修改属性后它自然而然会触发依赖更新的，所以在这里我们只需要关心两种操作即可，就是添加属性和删除属性，因为只有这两种操作会引起 keys 的变化，根据这个思路，我们来继续实现：

```ts
  set(target: Target, key: string | symbol, value: any, receiver: object) {
    const hadKey = hasOwn(target, key)
    // 设置value
    const result = Reflect.set(target, key, value, receiver)
    // 通知更新
    trigger(
      target,
      hadKey ? TriggerOpTypes.SET : TriggerOpTypes.ADD,
      key,
      value
    )
    return result
  },
  deleteProperty(target: Target, key: string | symbol) {
    // 判断要删除的key是否存在
    const hadKey = hasOwn(target, key)
    // 执行删除操作
    const result = Reflect.deleteProperty(target, key)
    // 只在存在key并且删除成功时再通知更新
    if (hadKey && result) {
      trigger(target, TriggerOpTypes.DELETE, key, undefined)
    }
    return result
  }
```

这里对之前的`set`和`deleteProperty`做了些修改，在 set 中判断了下 key 是否已经存在(`hadKey`)，然后根据这个判断是新增还是修改，这里的`trigger`和之前的`track`一样，也是加了个 type 参数；在`deleteProperty`中就是传了个`TriggerOpTypes.DELETE`表明是个删除操作，接下来再修改下之前写的 trigger 函数：

```ts
// 通知更新
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key: any,
  newValue?: any
) {
  // 获取该对象的depsMap
  const depsMap = targetMap.get(target)
  // 获取不到时说明没有触发过getter
  if (!depsMap) {
    return
  }
  const effects = new Set<ReactiveEffect>()
  const add = (data: Set<ReactiveEffect> | undefined) => {
    if (data) {
      data.forEach((effect) => {
        if (effect !== activeEffect) {
          effects.add(effect)
        }
      })
    }
  }
  add(depsMap.get(key))
  if (type === TriggerOpTypes.ADD || type === TriggerOpTypes.DELETE) {
    add(depsMap.get(ITERATE_KEY))
  }
  // 然后根据key获取deps，也就是之前存的effect函数
  // 执行所有的effect函数
  effects.forEach((effect) => {
    effect()
  })
}
```

这个改的比较多，最主要的是其中这一段代码：

```ts
if (type === TriggerOpTypes.ADD || type === TriggerOpTypes.DELETE) {
  add(depsMap.get(ITERATE_KEY))
}
```

就是判断下这个操作是添加或者删除的时候，就添加`depsMap.get(ITERATE_KEY)`，这个就是之前在`ownKeys`里面收集的依赖了，这样我们就有了`Object.keys`的收集依赖=>触发依赖的流程，你可以再试下文章开头的例子：在添加了 c 属性后，effect 函数成功地执行了。

#### 数组的边界情况

现在再试下另外一种情况

```ts
const state = reactive({ counts: [1, 2, 3] })

effect(() => {
  console.log(state.counts.length)
})

state.counts[3] = 4
```

这种情况并不会触发 effect，这是因为这里的 effect 函数只收集了`counts`的`length`，而我们修改时是通过`3`这个属性去修改的，这样当然不会触发依赖更新了，其实改的话很简单：

```ts
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key: any,
  newValue?: any
) {
  // ...
  if (type === TriggerOpTypes.ADD || type === TriggerOpTypes.DELETE) {
    add(depsMap.get(Array.isArray(target) ? 'length' : ITERATE_KEY))
  }
  // ...
}
```

因为数据也可以看作是一个对象，它的下标就是 key，之前的 ADD 和 DELETE 同样适用于数组，这样在这里只需要判断下是否是数组，是数组的话就从`length`这个 key 中取依赖，否则说明它是个对象，就还从`ITERATE_KEY`中取，这样修改之后，再运行上面那个例子，就能成功达到我们想要的效果了。

#### has

我们再来看下另外一种情况

```ts
const state = reactive({ counts: { a: 1, b: 2 } })

effect(() => {
  console.log(Reflect.has(state.counts, 'a'))
})

Reflect.deleteProperty(state.counts, 'a')
```

这个例子同样没有达到我们想要的效果，其实这是因为我们也没有拦截到操作，所以就没有收集依赖。在 Proxy 的 handlers 中还有另外一个属性就是`has`，它可以帮我们拦截到`in`操作符，这里的`Reflect.has`也会被拦截。下面我们来使用`has`改下我们的代码。

```ts
// handlers.ts

export const baseHandlers: ProxyHandler<object> = {
  get() {
    //...
  },
  set() {
    //...
  },
  deleteProperty() {
    //...
  },
  ownKeys() {
    //...
  },
  has(target: object, key: string | symbol): boolean {
    const result = Reflect.has(target, key)
    track(target, TrackOpTypes.HAS, key)
    return result
  }
}
```

新加了一个 has，其主要就是 track 触发收集就可以了，现在上面的例子就可以成功运行了。

#### 总结

这篇文章主要是对之前的代码增加了一些边界情况的判断，这些边界情况是经常能碰到的，其实除了这些还有很多的边界情况需要处理，比如还有数组的一些方法、Map、Set，还有一些值的类型的判断等，我这里就不写了，如果你感兴趣的话，你可以去`@vue/reactivity`这个库里面看，我这两篇文章也是参考这个库去实现的，虽然不如他这个库复杂，但是基本原理是一样的。

其实如果你认真研究下，就会发现这种响应式原理并不复杂，里面并没有什么神奇的魔法，都是围绕着一个 Proxy 去展开的，看过这篇文章后，相信你也能实现一个自己的响应式库。
