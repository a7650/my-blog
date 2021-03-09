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

通过编译后的代码可以发现，**旧的写法是将插槽内容作为children渲染的，会在父组件的渲染函数中创建，插槽内容的依赖会被父组件收集（name的dep收集到父组件的渲染watcher），而新的写法将插槽内容放在了scopedSlots中，会在子组件的渲染函数中调用，插槽内容的依赖会被子组件收集（name的dep收集到子组件的渲染watcher）**，最终导致的结果就是：当我们修改name这个属性时，旧的写法是调用父组件的更新（调用父组件的渲染watcher），然后在父组件更新过程中调用子组件更新（prePatch => updateChildComponent），而新的写法则是直接调用子组件的更新（调用子组件的渲染watcher）。

这样一来，旧的写法在更新时就多了一个父组件更新的过程，而新的写法由于直接更新子组件，就会更加高效，性能更好，所以推荐始终使用`v-slot:slotName`语法。

