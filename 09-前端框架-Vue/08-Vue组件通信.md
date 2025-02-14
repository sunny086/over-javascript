# 08-Vue 组件通信

## 一 props 实现父子组件通信

### 1.1 props 方式实现父传子

props 是 Vue 的常见选项对象之一，用于父子关系组件的数据传递。

父组件向子组件传递数据：

```html
<template>
  <!-- 传递静态数据 -->
  <Son name="lisi" />

  <!-- 传递动态数据 -->
  <Son :age="data.age" />
</template>

<script>
  export default {
    data() {
      return {
        age: 30,
      }
    },
  }
</script>
```

子组件接收数据：

```html
<script>
  export default {
    props: [name, age],
  }
</script>
```

### 1.2 props 实现子传父

通过传递函数可以实现子传父。

父组件传递函数给子组件：

```html
<template>
  <Son :getSonName="getSonName" />
</template>

<script>
  // 内部定义该函数
  export default {
    methods: {
      getSonName(data) {
        console.log(data)
      },
    },
  }
</script>
```

子组件接收函数，并传递数据：

```html
<template>
  <button @click="sendData">传递数据给父组件</button>
</template>

<script>
  // 内部定义该函数
  export default {
    props: ['getSonName']
    methods: {
      sendData() {
        this.getSonName('zs')
      },
    },
  }
</script>
```

### 1.3 data 与 props 对比

- data 中一般放置请求到的数据，props 里面放置的是传递过来的数据。
- data 中的数据都是可读写的，而 props 中的数据是只读的

贴士：在 props 中使用驼峰形式，模板中需要使用短横线拼接的形式，但字符串模板没有这个限制。

```html
<!-- props: userInfo -->
<template>
  <div>
    <h2 :user-info>{{userInfo}}</h2>
  </div>
</template>
```

### 1.4 props 数据类型校验

数据在传递时还可以提前进行数据类型校验，以提升应用的安全性：

```js
props: {
    sonMsg: String,
    isMan: Boolean
},
```

类型校验时可以添加默认值：

```js
props: {
  name: {
    type: String,
    default: 'lisi',
    required: true
  }
  movies: {
    type: Array,
    default(){    // 对象与数组类型的默认值必须是一个工厂函数
      return []
    }
  }
}
```

## 二 自定义事件实现数据传递

### 2.1 实现步骤

父组件中给子组将绑定事件：

```html
<template>
  <!-- 给子组件绑定自定义事件 showname -->
  <Son @show="getSonName" />
</template>

<script>
  export default {
    methods: {
      getSonName(data) {
        console.log(data)
      },
    },
  }
</script>
```

子组件触发事件，这里无需使用 props 接收了，由`$emit`触发：

```html
<template>
  <button @click="sendData">传递数据给父组件</button>
</template>

<script>
  export default {
    methods: {
      sendData() {
        this.$emit('show', 'lisi')
      },
    },
  }
</script>
```

### 2.2 解绑自定义事件

```html
<template>
  <button @click="unBind">解绑</button>
</template>

<script>
  export default {
    methods: {
      unBind() {
        // 解绑多个参数为 ['show', 'post']
        this.$off('show')
      },
    },
  }
</script>
```

注意：销毁组件会移除该组件实例上的成员，自定义事件也会因此不再生效。

## 三 不推荐的父子通信方式

### 3.1 ref 实现父访问子

ref 可以获取到原生节点，用于在父组件中直接操作子组件：

```html
<template>
  <input ref="mytext" />
  <button @click="getData">传递数据给父组件</button>
</template>

<script>
  export default {
    methods: {
      getData() {
        console.log(this.$refs.mytext.value)
      },
    },
  }
</script>
```

ref 通信方式需要在视图结构上添加 ref，不太推荐，但是灵活性强，与自定义事件结合使用如下：

父组件定义 ref：

```html
<template>
  <Son ref="mySon" />
</template>

<script>
  export default {
    mounted() {
      this.$refs.mySon.$on('show', function (data) {
        console.log(data)
      })
    },
  }
</script>
```

子组件传递数据：

```html
<template>
  <button @click="sendData">传递数据给父组件</button>
</template>

<script>
  export default {
    methods: {
      sendData() {
        this.$emit('show', 'lisi')
      },
    },
  }
</script>
```

灵活性的体现示例：可以定时设定接收事件触发。

### 3.2 parent 实现子访问父

`this.$parent` 是当前组件的父组件，该方法也不推荐使用。

## 四 事件总线实现组件通信

组件之间一般都存在着联系，可以通过祖先组件层级关系进行互相通信，由于该方式需要层层跨越，实现起来也比较麻烦。

Vue 为非父子关系组件提供了事件总线机制来实现通信。事件总线独立于所有组件之外，其本质就是一个空的 Vue 实例。

贴士：事件总线的操作类似在 window 上添加一个属性，这样所有地方就可以访问到该属性，同理在 Vue 上添加的属性，在该 Vue 产生的实例上也都可以访问到。

```html
<script>
  let bus = new Vue() // 新建中央事件总线

  Vue.component('publish', {
    template: `
                <div>
                    <input type="text" />
                    <button @click="handleClick()">发布</button>
                </div>`,
    methods: {
      handleClick() {
        bus.$emit('msg', 'helloworld')
      },
    },
  })

  Vue.component('subcribe', {
    template: `
                <div>
                    数据为：
                </div>`,
    mounted() {
      // 这里让组件尽早的订阅总线消息
      bus.$on('msg', (data) => {
        alert(data)
      })
    },
  })

  let app = new Vue({
    el: '#app',
    data: {},
  })
</script>
```

一般使用全局事件总线，定义方式如下：

```js
// main.js
new Vue({
  el: '#app',
  render: (h) => h(App),
  beforeCreate() {
    Vue.prototype.$bus = this // 定义全局事件总线
  },
})
```

使用全局事件总线发送数据：

```js
methods: {
  sendData(){
    this.$bus.$emit('hi', 'Lisi')
  }
}
```

使用全局事件总线接收数据、销毁事件：

```js
mounted(){
  this.$bus.$on('hi', data => {})
},
beforeDestroy(){
  this.$bus.$off('hi')
}
```

## 五 发布订阅实现组件通信

发布订阅借助于第三方库：`pubsub-js`。

发送方：

```js
import pubsub from 'pubsub-js'
const sendMsg = () => {
  pubsub.publish('hi', { name: 'lisi', age: 30 })
}
```

接收方：

```js
import pubsub from 'pubsub-js'

let pubID = null

// 获取订阅
const getMsg = () => {
  pubID = pubsub.subscribe('hi', function (msgName, data) {
    // 执行 hi 消息的回调：msgName 是消息名，data 是数据
  })
}

// 取消订阅
const unSub = () => {
  pubsub.unsubscribe(pubID)
}
```
