# 12-Vue 插件编写

## 一 Vue 插件编写方式

Vue 插件用于扩展 Vue，有两种方式自定义插件。

方式一：

```js
// 直接在 prototype 身上定义
Vue.prototype.$aaa = '这是定义的插件'
```

方式二：

```js
// 定义一个对象为插件，然后在 use() 里调用
    let obj = {
        // 必须有一个 install 函数，由 vue 调用
        install:function(Vue,opt){
            Vue.prototype.$aaa = '这是定义的插件';
        }
    }

    Vue.use(obj, {a:1})；
```

## 二 示例

案列：获取和设置 localStorage 的存储

```js
// 对象文件：utils.js
let local = {
  save(key, value) {
    localStorage.setItem(key, JSON.stringify(value))
  },
  fetch() {
    return JSON.parse(localStorage.getItem(key) || {})
  },
}

export default {
  install: function (vm) {
    vm.prototype.$local = local // local 对象挂载到 Vue 原型上
  },
}
```

```js
    // 使用 utils 插件的文件：

    import Utile from './lib/utils';

    Vue.use(Utile)；

    // 一旦作为一个插件使用之后，就可以在每个组件里面通过 this 访问到 local 对象了
```

**可以用一个文件写很多的对象，把想要暴露出去的对象挂载到 Vue 原型身上就行；如：**

```js
let obj1 = {}
let obj2 = {}
let obj3 = {}

export default {
  install: function (vm) {
    vm.prototype.$obj1 = obj1
    vm.prototype.$obj2 = obj2
    vm.prototype.$obj3 = obj3
  },
}
```
