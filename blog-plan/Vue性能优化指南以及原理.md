### 使用`v-slot:slotName`，而不是`slot="slotName"`

`v-slot`是 2.6 新增的语法，具体可查看:[Vue2.6](https://zhuanlan.zhihu.com/p/56260917)，虽然 2.6 发布已经是快两年前的事情了，但是现在仍然有不少人仍然在使用`slot="slotName"`这个语法，虽然这两个语法都能达到相同的效果，但是内部的逻辑确实不一样的，我们先来看下这两种语法分别会被编译成什么：

使用新的写法，对于父组件中的以下模板：

```html
<child>
  <template v-slot:name>{{name}}</template>
</child>
```

会被编译成：

```js
function render() {
  with (this) {
    return _c('child', {
      scopedSlots: _u([
        {
          key: 'name',
          fn: function () {
            return [_v(_s(name))]
          },
          proxy: true
        }
      ])
    })
  }
}
```

使用旧的写法，对于以下模板：

```html
<child>
  <template slot="name">{{name}}</template>
</child>
```

会被编译成：

```js
function render() {
  with (this) {
    return _c(
      'child',
      [
        _c(
          'template',
          {
            slot: 'name'
          },
          [_v(_s(name))]
        )
      ],
      2
    )
  }
}
```

通过编译后的代码可以发现，**旧的写法是将插槽内容作为 children 渲染的，会在父组件的渲染函数中创建，插槽内容的依赖会被父组件收集（name 的 dep 收集到父组件的渲染 watcher），而新的写法将插槽内容放在了 scopedSlots 中，会在子组件的渲染函数中调用，插槽内容的依赖会被子组件收集（name 的 dep 收集到子组件的渲染 watcher）**，最终导致的结果就是：当我们修改 name 这个属性时，旧的写法是调用父组件的更新（调用父组件的渲染 watcher），然后在父组件更新过程中调用子组件更新（prePatch => updateChildComponent），而新的写法则是直接调用子组件的更新（调用子组件的渲染 watcher）。

这样一来，旧的写法在更新时就多了一个父组件更新的过程，而新的写法由于直接更新子组件，就会更加高效，性能更好，所以推荐始终使用`v-slot:slotName`语法。

### 使用计算属性

这一点已经被提及很多次了，计算属性最大的一个特点就是它是可以被缓存的，这个缓存指的是只要他的依赖的不发生改变，它就不会被重新求值，再次访问时会直接拿到缓存的值，在做一些复杂的计算时，可以极大提升性能。可以看以下代码：

```html
<template>
  <div>{{superCount}}</div>
</template>
<script>
  export default {
    data() {
      return {
        count: 1
      }
    },
    computed: {
      superCount() {
        let superCount = this.count
        // 假设这里有个复杂的计算
        for (let i = 0; i < 10000; i++) {
          superCount++
        }
        return superCount
      }
    },
    created() {
      console.log(this.superCount)
    },
    mounted() {
      console.log(this.superCount)
    }
  }
</script>
```

这个例子中，在 created、mounted 以及模板中都访问了 superCount 属性，这三次访问中，实际上只有第一次即`created`时才会对 superCount 求值，由于 count 属性并未改变，其余两次都是直接返回缓存的 value，对于计算属性更加详细的介绍可以看我之前写的文章：[Vue computed 是如何实现的？](https://juejin.cn/post/6925013384069038088)。

### 使用函数式组件

对于某些组件，如果我们只是用来显示一些数据，不需要管理状态，监听数据等，那么就可以用函数式组件。函数式组件是无状态的，无实例的，这意味着在初始化时不需要初始化状态，不需要创建实例，也不需要去处理生命周期等，相比有状态组件，会更加轻量，同时性能也更好。具体的函数式组件使用方式可参考官方文档：[函数式组件](https://cn.vuejs.org/v2/guide/render-function.html#%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BB%84%E4%BB%B6)

我们可以写一个简单的 demo 来验证下这个优化：

```html
// UserProfile.vue
<template>
  <div class="user-profile">{{ name }}</div>
</template>

<script>
  export default {
    props: ['name'],
    data() {
      return {}
    },
    methods: {}
  }
</script>
<style scoped></style>

// App.vue
<template>
  <div id="app">
    <UserProfile v-for="item in list" :key="item" :name="item" />
  </div>
</template>

<script>
  import UserProfile from './components/UserProfile'

  export default {
    name: 'App',
    components: { UserProfile },
    data() {
      return {
        list: Array(500)
          .fill(null)
          .map((_, idx) => 'Test' + idx)
      }
    },
    beforeMount() {
      this.start = Date.now()
    },
    mounted() {
      console.log('用时:', Date.now() - this.start)
    }
  }
</script>

<style></style>
```

UserProfile 这个组件只渲染了 props 的 name，然后在 App.vue 中调用 500 次，统计从 beforeMount 到 mounted 的耗时，即为 500 个子组件（UserProfile）初始化的耗时。

经过我多次尝试后，发现耗时一直在 30ms 左右，那么现在我们再把改成 UserProfile 改成函数式组件：

```html
<template functional>
  <div class="user-profile">{{ props.name }}</div>
</template>
```

此时再经过多次尝试后，初始化的耗时一直在 10-15ms，这些足以说明函数式组件比有状态组件有着更好的性能。

### 结合场景使用 v-show 和 v-if

以下是两个使用 v-show 和 v-if 的模板

```html
<template>
  <div>
    <UserProfile :user="user1" v-if="visible" />
    <button @click="visible = !visible">toggle</button>
  </div>
</template>
```

```html
<template>
  <div>
    <UserProfile :user="user1" v-show="visible" />
    <button @click="visible = !visible">toggle</button>
  </div>
</template>
```

这两者的作用都是用来控制某些组件或 DOM 的显示/隐藏，在讨论它们的性能差异之前，先来分析下这两者有何不同。其中，v-if 的模板会被编译成：

```js
function render() {
  with (this) {
    return _c(
      'div',
      [
        visible
          ? _c('UserProfile', {
              attrs: {
                user: user1
              }
            })
          : _e(),
        _c(
          'button',
          {
            on: {
              click: function ($event) {
                visible = !visible
              }
            }
          },
          [_v('toggle')]
        )
      ],
      1
    )
  }
}
```

可以看到，v-if 的部分被转换成了一个三元表达式，visible 为 true 时，创建一个 UserProfile 的 VNode，否则创建一个空 VNode，在 patch 的时候，新旧节点不一样，就会移除旧的节点或创建新的节点，这样的话`UserProfile`也会跟着销毁/创建。如果`UserProfile`组件里有很多 DOM，或者要执行很多初始化/销毁逻辑，那么随着 visible 的切换，势必会浪费掉很多性能。这个时候就可以用 v-show 进行优化，我们来看下 v-show 编译后的代码：

```js
function render() {
  with (this) {
    return _c(
      'div',
      [
        _c('UserProfile', {
          directives: [
            {
              name: 'show',
              rawName: 'v-show',
              value: visible,
              expression: 'visible'
            }
          ],
          attrs: {
            user: user1
          }
        }),
        _c(
          'button',
          {
            on: {
              click: function ($event) {
                visible = !visible
              }
            }
          },
          [_v('toggle')]
        )
      ],
      1
    )
  }
}
```

实际上，v-show 是一个 Vue 内部的指令，在这个指令的代码中，主要执行了以下逻辑：

```js
el.style.display = value ? el.__vOriginalDisplay : 'none'
```

它其实是通过切换元素的 display 属性来控制的，和 v-if 相比，不需要在 patch 阶段创建/移除节点，只是根据`v-show`上绑定的值来控制 DOM 元素的`style.display`属性，在频繁切换的场景下就可以节省很多性能。

但是并不是说`v-show`可以在任何情况下都替换`v-if`，如果初始值是`false`时，`v-if`并不会创建隐藏的节点，但是`v-show`会创建，并通过设置`style.display='none'`来隐藏，虽然外表看上去这个 DOM 都是被隐藏的，但是`v-show`已经完整的走了一遍创建的流程，造成了性能的浪费。

所以，`v-if`的优势体现在初始化时，`v-show`体现在更新时，当然并不是要求你绝对按照这个方式来，比如某些组件初始化时会请求数据，而你想先隐藏组件，然后在显示时能立刻看到数据，这时候就可以用`v-show`，又或者你想每次显示这个组件时都是最新的数据，那么你就可以用`v-if`，所以我们要结合具体业务场景去选一个合适的方式。

### 使用 keep-alive

在动态组件的场景下：

```html
<template>
  <div>
    <component :is="currentComponent" />
  </div>
</template>
```

这个时候有多个组件来回切换，`currentComponent`每变一次，相关的组件就会销毁/创建一次，如果这些组件比较复杂的话，就会造成一定的性能压力，其实我们可以使用 keep-alive 将这些组件缓存起来：

```html
<template>
  <div>
    <keep-alive>
      <component :is="currentComponent" />
    </keep-alive>
  </div>
</template>
```

`keep-alive`的作用就是将它包裹的组件在第一次渲染后就缓存起来，下次需要时就直接从缓存里面取，避免了不必要的性能浪费，在讨论上个问题时，说的是`v-show`初始时性能压力大，因为它要创建所有的组件，其实可以用`keep-alive`优化下：

```js
<template>
  <div>
    <keep-alive>
      <UserProfileA v-if="visible" />
      <UserProfileB v-else />
    </keep-alive>
  </div>
</template>
```

这样的话，初始化时不会渲染`UserProfileB`组件，当切换`visible`时，才会渲染`UserProfileB`组件，同时被`keep-alive`缓存下来，频繁切换时，由于是直接从缓存中取，所以会节省很多性能，所以这种方式在初始化和更新时都有较好的性能。

但是`keep-alive`并不是没有缺点，组件被缓存时会占用内存，属于空间和时间上的取舍，在实际开发中要根据场景选择合适的方式。

### 避免 v-for 和 v-if 同时使用

这一点是 Vue 官方的风格指南中明确指出的一点：[Vue 风格指南](https://cn.vuejs.org/v2/style-guide/#%E9%81%BF%E5%85%8D-v-if-%E5%92%8C-v-for-%E7%94%A8%E5%9C%A8%E4%B8%80%E8%B5%B7%E5%BF%85%E8%A6%81)

如以下模板：

```html
<ul>
  <li v-for="user in users" v-if="user.isActive" :key="user.id">
    {{ user.name }}
  </li>
</ul>
```

会被编译成：

```js
// 简化版
function render() {
  return _c(
    'ul',
    this.users.map((user) => {
      return user.isActive
        ? _c(
            'li',
            {
              key: user.id
            },
            [_v(_s(user.name))]
          )
        : _e()
    }),
    0
  )
}
```

可以看到，这里是先遍历（v-for），再判断（v-if），这里有个问题就是：如果你有一万条数据，其中只有 100 条是`isActive`状态的，你只希望显示这100条，但是实际在渲染时，每一次渲染，这一万条数据都会被遍历一遍。比如你在这个组件内的其他地方改变了某个响应式数据时，会触发重新渲染，调用渲染函数，调用渲染函数时，就会执行到上面的代码，从而将这一万条数据遍历一遍，即使你的`users`没有发生任何改变。

为了避免这种性能损失，我们可以用计算属性代替：
