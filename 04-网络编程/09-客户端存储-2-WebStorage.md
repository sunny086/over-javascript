# 09-客户端存储 -2-WebStorage

## 一 Web Storage 概念

Web Storage 目的是解决通过客户端存储不需要频繁发送回服务器的数据时使用 cookie 的问题。

Web Storage 的第 2 版定义了两个对象：

- localStorage：永久存储机制
- sessionStorage：跨会话的存储机制

这两种浏览器存储 API 提供了在浏览器中不受页面刷新影响而存储数据的两种方式。2009 年之后，该 API 被大多浏览器接受。

## 二 API

### 2.0 Storage 类型

Storage 类型用于保存名/值对数据，直至存储空间上限（由浏览器决定）。Storage 的实例与其他对象一样，但增加了以下方法：

```txt
clear()：删除所有值；不在 Firefox 中实现。
getItem(name)：取得给定 name 的值。
key(index)：取得给定数值位置的名称。
removeItem(name)：删除给定 name 的名/值对。
setItem(name, value)：设置给定 name 的值
```

getItem()、removeItem() 和 setItem() 方法可以直接或间接通过 Storage 对象调用。因为每个数据项都作为属性存储在该对象上，所以可以使用点或方括号操作符访问这些属性，通过同样的操作来设置值，也可以使用 delete 操作符删除属性。即便如此，通常还是建议使用方法而非属性来执行这些操作，以免意外重写某个已存在的对象成员。

通过 length 属性可以确定 Storage 对象中保存了多少名/值对。我们无法确定对象中所有数据占用的空间大小，尽管 IE8 提供了 remainingSpace 属性，用于确定还有多少存储空间（以字节计）可用。

### 2.1 sessionStorage

sessionStorage 对象只存储会话数据，这意味着数据只会存储到浏览器关闭。这跟浏览器关闭时会消失的会话 cookie 类似。存储在 sessionStorage 中的数据不受页面刷新影响，可以在浏览器崩溃并重启后恢复。（取决于浏览器，Firefox 和 WebKit 支持，IE 不支持。）

因为 sessionStorage 对象与服务器会话紧密相关，所以在运行本地文件时不能使用。存储在 sessionStorage 对象中的数据只能由最初存储数据的页面使用，在多页应用程序中的用处有限。

因为 sessionStorage 对象是 Storage 的实例，所以可以通过使用 setItem() 方法或直接给属性赋值给它添加数据。下面是使用这两种方式的例子：

```js
// 使用方法存储数据
sessionStorage.setItem('name', 'Nicholas')
// 使用属性存储数据
sessionStorage.book = 'Professional JavaScript'
```

所有现代浏览器在实现存储写入时都使用了同步阻塞方式，因此数据会被立即提交到存储。具体 API 的实现可能不会立即把数据写入磁盘（而是使用某种不同的物理存储），但这个区别在 JavaScript 层面是不可见的。通过 Web Storage 写入的任何数据都可以立即被读取。

老版 IE 以异步方式实现了数据写入，因此给数据赋值的时间和数据写入磁盘的时间可能存在延迟。对于少量数据，这里的差别可以忽略不计，但对于大量数据，就可以注意到 IE 中 JavaScript 恢复执行的速度比其他浏览器更快。这是因为实际写入磁盘的进程被转移了。在 IE8 中可以在数据赋值前调用 begin()、之后调用 commit() 来强制将数据写入磁盘。比如：

```js
// 仅适用于 IE8
sessionStorage.begin()
sessionStorage.id = '1001'
sessionStorage.name = 'Lisi'
sessionStorage.commit()
```

以上代码确保了"name"和"book"在 commit() 调用之后会立即写入磁盘。调用 begin() 是为了保证在代码执行期间不会有写入磁盘的操作。对于少量数据，这个过程不是必要的，但对于较大的数据量，如文档，则可以考虑使用这种事务性方法。

对存在于 sessionStorage 上的数据，可以使用 getItem() 或直接访问属性名来取得：

```js
// 使用方法取得数据
let name = sessionStorage.getItem("name");
// 使用属性取得数据
let book = sessionStorage.book;
可以结合 sessionStorage 的 length 属性和 key()方法遍历所有的值：
for (let i = 0, len = sessionStorage.length; i < len; i++){
let key = sessionStorage.key(i);
let value = sessionStorage.getItem(key);
alert(`${key}=`${value}`);
}
```

这里通过 key() 先取得给定位置中的数据名称，然后使用该名称通过 getItem() 取得值，可以依次访问 sessionStorage 中的名/值对。也可以使用 for-in 循环迭代 sessionStorage 的值：

```js
for (let key in sessionStorage) {
  let value = sessionStorage.getItem(key)
  alert(`${key}=${value}`)
}
```

每次循环，key 都会被赋予 sessionStorage 中的一个名称；这里不会返回内置方法或 length 属性。

要从 sessionStorage 中删除数据，可以使用 delete 操作符直接删除对象属性，也可以使用 removeItem() 方法。下面是使用这两种方式的例子：

```js
// 使用 delete 删除值
delete sessionStorage.name
// 使用方法删除值
sessionStorage.removeItem('book')
```

sessionStorage 对象应该主要用于存储只在会话期间有效的小块数据。如果需要跨会话持久存储
数据，可以使用 globalStorage 或 localStorage。

### 2.2 localStorage

在修订的 HTML5 规范里，localStorage 对象取代了 globalStorage，作为在客户端持久存储数据的机制。要访问同一个 localStorage 对象，页面必须来自同一个域（子域不可以）、在相同的端口上使用相同的协议。

因为 localStorage 是 Storage 的实例，所以可以像使用 sessionStorage 一样使用 localStorage。比如下面这几个例子：

```js
// 使用方法存储数据
localStorage.setItem('name', 'Nicholas')
// 使用属性存储数据
localStorage.book = 'Professional JavaScript'
// 使用方法取得数据
let name = localStorage.getItem('name')
// 使用属性取得数据
let book = localStorage.book
```

两种存储方法的区别在于，存储在 localStorage 中的数据会保留到通过 JavaScript 删除或者用户清除浏览器缓存。localStorage 数据不受页面刷新影响，也不会因关闭窗口、标签页或重新启动浏览器而丢失。

## 三 存储事件

每当 Storage 对象发生变化时，都会在文档上触发 storage 事件。使用属性或 setItem() 设置
值、使用 delete 或 removeItem() 删除值，以及每次调用 clear() 时都会触发这个事件。这个事件的
事件对象有如下 4 个属性：

```txt
domain：存储变化对应的域。
key：被设置或删除的键。
newValue：键被设置的新值，若键被删除则为 null。
oldValue：键变化之前的值。
```

可以使用如下代码监听 storage 事件：

```js
window.addEventListener('storage', (event) =>
  alert('Storage changed for ${event.domain}')
)
```

对于 sessionStorage 和 localStorage 上的任何更改都会触发 storage 事件，但 storage 事件不会区分这两者。

## 四 限制

与其他客户端数据存储方案一样，Web Storage 也有限制。具体的限制取决于特定的浏览器。一般来说，客户端数据的大小限制是按照每个源（协议、域和端口）来设置的，因此每个源有固定大小的数据存储空间。分析存储数据的页面的源可以加强这一限制。

不同浏览器给 localStorage 和 sessionStorage 设置了不同的空间限制，但大多数会限制为每个源 5MB。
