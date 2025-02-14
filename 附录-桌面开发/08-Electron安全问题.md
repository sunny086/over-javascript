# Electron 应用安全问题

## 一 常见保护应用手法

### 1.1 生产环境关闭开发者工具

```js
new BrowserWindow({
  // 其他配置
  webPreferces: {
    devTools: false,
  },
})
```

上述方式会让任何手段都无法打开开发者工具。而在传统 web 项目中，则会变得复杂，需要禁用 F12，禁用右键，禁用 Ctrl+Shift+I。

### 1.2 源码压缩与混淆

常用的压缩工具为：uglify-js<https://github.com/mishoo/UglifyJS>

Webpack 同时也自带了压缩功能。

Electron 提供了工具 asar 用来保护前端源码。在 Electron 开发者工具控制台中输入 `__filename` 就可以看到资源目录，但是在资源目录却只有 `.asar` 的文件（非文件夹）。Electron 无需解压该文件即可读取文件内容，只有在一些特殊场景，如处理依赖真实文件路径的底层系统方法时，才会解压，如：`child_process.execFile`。

此外，该库还能解决 Win 下路径名过长的问题。

使用方式：

```txt
npm i asar -D
npx asar pack your-app app.asar
```

对一些核心代码，我们可以考虑将其直接编译为 V8 字节码文件，工具为：<https://github.com/OsamaAbbas/bytenode>

## 二 客户信息保护

### 2.1 禁用 Node.js 集成

如果在 webContent 中加载的是不受控的内容，必须要禁用 Node 集成，因为会造成第三方利用 Node 访问到操作系统。

涉及的相关配置位于创建 BrowserWindow、当前 BrowserView 加载第三方内容、webview 标签加载第三方内容。

### 2.2 启用同源策略

与集成 Node 的安全限制相同，在加载不受信 内容时，也应该 **启用同源策略**

### 2.3 启用沙箱环境

如果想让 Electron 客户端拥有与 web 系统一样的安全沙箱环境，无需控制 webecurity、node 集成等，则在创建窗口、webview 时，可以启用沙箱隔离特性：

```js
let win = new BrowserWindow({
  webPreferences: {
    sandbox: true,
  },
})
```

### 2.4 禁用 webview

在 Electron 项目中，BrowserView 默认是不能使用 webview 标签的，除非在创建窗口时启用 `webviewTag:true`, 但是并不推荐这样做，因为大多数场景中 webview 可以使用 BrowserView 代替。

一旦开启该属性，webview 内部的第三方内容可以在其自身内部创建一个 webview 标签，此时会相继引发不安全问题。为了解决该问题，需要监听 webContents 的 will-attach-webview 事件：

```js
app.on('web-contents-created', (event, contents) => {
  // 一旦有 webview 被创建则触发该事件
  contents.on('will-attach-webview', (event, webPreferences, params) => {
    delete webPreferences.preload
    delete webPreferences.preloadURL
    webPreferences.nodeIntegration = false
    if (!params.src.startsWith('https://demo.com/')) {
      event.preventDefault()
    }
  })
})
```

## 三 网络保护

### 3.1 虚假证书

可以使用 session 模块提升与服务端交互的安全性，可以在一定程度上屏蔽恶意用户同归伪造客户端证书来分析 Electron 应用的网络数据：

```js
let session = win.webContents.session

// 当前会话的钩子函数
session.setCertificateVerifyProc((request, callback) => {
  if (request.certificate.issuer.commonName == 'DO_NOT_TRUST_FiddlerRoot') {
    // 假设现在安装的是 Fiddler 的证书
    callback(-2) // 如果不符合预期，传入 -2 驳回
  } else {
    callback(-3) // -3 表示使用 Chromium 的验证结果，0 表示成功并禁止使用证书透明度验证
  }
})
```

### 3.2 防盗链

一般情况下，采用识别请求头 Referer 方式防盗链，那么如果我们还要为本地 Electron 应用提供静态服务，就需要如下设置：

```js
let session = win.webContents.session

let requestFilter = {
  urls: ['http://*/*', 'https://*/*'],
}

session.webRequest.onBeforeSendHeaders(requestFilter, (details, callback) => {
  if (details.resourceType == 'image' && details.method == 'GET') {
    delete details.requestHeades['Referer']
  }
  callback({
    requestHeades: details.requestHeades6t,
  })
})
```

### 3.3 数据加密

可以利用 Node 的数据加密功能对一些敏感数据进行加密后传输。

electron-store 模块内置了加密解密功能：

```js
const Store = require('electron-store')

const schema = {
  foo: {
    type: 'number',
    default: 100,
  },
}

const store = new Store({ schema, encryptionKey: 'myKey' })
```

### 3.4 屏幕保护

窗口的鼠标等操作可以被黑客截货，保护方式：

```js
win.setContentProtection('true') // 当使用工具捕捉窗口时，会显示黑色区域
```

## 四 稳定性

### 4.1 全局异常捕获

Electron 捕获全局异常方式：

```js
process.on('uncaughtException', (err, origin) => {
  // 执行日志收集，显示异常信息并重新加载界面
})
```

### 4.2 崩溃恢复

监听渲染进程的 crashed 事件可以知道渲染进程何时发生了崩溃：

```js
const { dialog } = require('electron')

win.webContents.on('crashed', async (e, killed) => {
  // 加入日志收集
  let result = await dialog.showMessageBox({
    type: 'error',
    title: '应用程序崩溃',
    message: '当前应用程序发生异常，是否退出？',
    buttons: ['是', '否'],
  })
})
```

模拟崩溃的方式：

```js
process.crash() // 模拟崩溃
process.hang() // 模拟挂起
```
