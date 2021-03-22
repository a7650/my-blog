### 前言

这篇文章主要来分享下我以前用 Vue3 写的一个组件，

项目地址 ： [Vue3DraggableResizable](https://github.com/a7650/vue3-draggable-resizable)

![logo (2).png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50d6c7ffea8b44afa5c756482ef15997~tplv-k3u1fbpfcp-zoom-1.image)

### 组件功能

- 支持拖拽和缩放，可分别定义开启或关闭

- 自定义缩放句柄（缩放时共有八个方位可操作，可分别定义开启或关闭）

- 可将组件的拖动和缩放限制在其父节点内

- 可自定义组件内各种类名

- 可配合内置的`DraggableContainer`组件方便地实现参考线以及自动吸附功能。

该组件所支持的参数和事件加起来有几十种，可进行各种配置，具体可查看[Github](https://github.com/a7650/vue3-draggable-resizable)的详细文档。

下面我来介绍下使用方式。

### 基本功能

首先要注册组件：

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import Vue3DraggableResizable from 'vue3-draggable-resizable'
//需引入默认样式
import 'vue3-draggable-resizable/dist/Vue3DraggableResizable.css'

// 你将会获得一个名为Vue3DraggableResizable的全局组件
createApp(App).use(Vue3DraggableResizable).mount('#app')
```

执行`use(Vue3DraggableResizable)`后将会全局注册两个组件：`Vue3DraggableResizable`和`DraggableContainer`，`Vue3DraggableResizable`是拖拽缩放用的组件，`DraggableContainer`可用来实现参考线和自动吸附。

注册后就可以使用了：

```html
<template>
  <div id="app">
    <div class="parent">
      <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
    </div>
  </div>
</template>

<script>
  import { defineComponent } from 'vue'

  export default defineComponent({
    data() {
      return {
        msg: 'Hello World. Hello World. Hello World.'
      }
    }
  })
</script>
<style lang="less" scoped>
  .parent {
    width: 300px;
    height: 300px;
    position: absolute;
    top: 100px;
    left: 200px;
    position: relative;
    border: 1px solid #000;
  }
</style>
```

很简单，直接使用即可，可以在`Vue3DraggableResizable`内放任何东西。

图[1]

也可以锁定比例，只需要传入`:lockAspectRatio="true"`参数就可以了：

```html
<Vue3DraggableResizable :lockAspectRatio="true">
  {{ msg }}
</Vue3DraggableResizable>
```

也可以让组件只在 X 轴上移动或只在 Y 轴上移动，传入`disabledX`或`disabledY`即可：

```html
<Vue3DraggableResizable :disabledX="true"> {{ msg }} </Vue3DraggableResizable>
```

可通过`parent`属性控制组件是否只能在其父节点内移动：

```html
<Vue3DraggableResizable :parent="true"> {{ msg }} </Vue3DraggableResizable>
```

[图 2]

除了我介绍的这几个，还有其他很多功能，感兴趣的话可以去 GitHub 上看详细的文档。

下面我介绍下参考线和吸附对齐的功能。

### 参考线、吸附对齐

这个功能需要配合`DraggableContainer`组件一起使用。

```html
<template>
  <div id="app">
    <div class="parent">
      <DraggableContainer>
        <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
        <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
      </DraggableContainer>
    </div>
  </div>
</template>

<script>
  import { defineComponent } from 'vue'

  export default defineComponent({
    data() {
      return {
        msg: 'Hello World. Hello World. Hello World.'
      }
    }
  })
</script>
<style lang="less" scoped>
  .parent {
    width: 300px;
    height: 300px;
    position: absolute;
    top: 100px;
    left: 200px;
    position: relative;
    border: 1px solid #000;
    .vdr-container {
      background-color: #999;
    }
  }
</style>
```

在刚才的基础上，直接 yong`DraggableContainer`组件套一下就可以了。其子组件`Vue3DraggableResizable`在移动时候就会自动吸附。

[图 3]

可以使用`adsorbParent`属性，使靠近父节点时也吸附对齐：

```html
<template>
  <div id="app">
    <div class="parent">
      <DraggableContainer adsorbParent>
        <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
        <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
      </DraggableContainer>
    </div>
  </div>
</template>
```

[图 4]

你也可以通过`adsorbCols`或`adsorbRows`自定义列或行的校准线，元素在靠近这些线时，会产生吸附，

```html
<DraggableContainer :adsorbCols="[10, 50, 100]" :adsorbParent="false">
  <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
</DraggableContainer>
```

[图 5]

也可以修改参考线颜色：

```html
<DraggableContainer :referenceLineColor="#0f0">
  <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
</DraggableContainer>
```

当然也可以不显示参考线：

```html
<DraggableContainer :referenceLineVisible="false">
  <Vue3DraggableResizable> {{ msg }} </Vue3DraggableResizable>
</DraggableContainer>
```

参考线虽然不显示，但是自动吸附仍然生效。

### 最后

这是去年写的一个项目，现在拿出来分享下，如果在使用中有什么问题的话，欢迎在评论区或者 GitHub 上和我交流。

---

看到这里就点个赞吧，感谢~~
