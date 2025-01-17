# Electron 与硬件交互

H5 虽然新增了对计算机硬件设备的访问，但其限制极多，Electron 却可以自由的使用这些技术，因为其 API 提供者为原生的 C++。

## 一 音视频设备

### 1.1 访问摄像头、麦克风

```js
// 该配置用来确定是否从流中获取音频、视频
let option = {
  audio: true,
  video: true,
  // width: 1280, height: 720,     // 视频大小
  // facingMode: "user"             // 获取前置摄像头，值 enviroment 表示后置摄像头
}

// 获取用户的音视频流
let mediaStream = await navigator.mediaDevices.getUserMedia(option)

videoDom.srcObject = mediaStream // videoDom 为 HTML 中的 video 标签
videoDom.onloadedmetadata = function (e) {
  video.play()
}
```

### 1.2 录屏

Electron desktopCapturer 模块可以用来获取桌面的屏幕视频流：

```js
const { desktopCapturer } = require('electron')

// 获取所有桌面信息，根据参数进行过滤
let sources = await desktopCapturer.getSources({
  types: ['window', 'screen'],
})

let target = sources.find((v) => v.name == '微信')

let mediaStream = await navigator.mediaDevices.getUserMedia({
  audio: false,
  video: {
    mandatory: {
      chromeMediaSource: 'desktop',
      chromeMediaSrouceId: target.id,
    },
  },
})
```

## 二 电源

电源模块可以使用 HTML5 的模块：

```js
let batterManager = await navigator.getBattery()
```

也可以使用 Electron 封装的模块：

```js
const { powerMonitor } = require('electron').remote

// 该模块可以用于监控系统是否挂起、恢复
powerMonitor.on('suspend', () => {
  console.log('中断...')
})

powerMonitor.on('resume', () => {
  console.log('恢复...')
})
```

## 三 打印机与导出 PDF

Electron 支持把 webContents 的内容发送至打印机：

```js
let { remote } = require('electron')

let webContents = remote.getCurrentWebContents()
webContents.print(
  {
    silent: false,
    printBackground: true,
    deviceName: '',
  },
  (success, erroType) => {}
)
```

如何找出打印机：

```js
let printers = webContents.getPrinters()
```

导出 PDF 示例：

```js
const { remote } = require('electron')
const path = require('path')
const fs = reuqire('fs')

let webContents = remote.getCurrentWebContents()
let data = await webContents.printToPDF({}) // 返回一个 Buffer 缓存
let filePaht = path.join(__static, 'demo.pdf')
fs.writeFile(filePath, data, (error) => {})
```

## 四 显示器与自助机

### 4.1 多显示器获取

Electron 获取主显示器、外接显示器信息，主显示器

```js
const { remote } = require('electron')
let mainScreen = remote.screen.getPrimaryDisplay() // 主显示器包括显示器 id，bounds 绑定区域，据此判断是否为外接
```

让窗口显示在外接显示器上：

```js
const { screen } = require('electron') // ready 事件后才可以使用

let displays = screen.getAllDisplay()
let externalDisplay = displays.find((display) => {
  return display.bounds.x !== 0 || display.bounds.y !== 0
})

if (externalDisplay) {
  win = new BrowserWindow({
    x: externalDisplay.bounds.x + 50,
    y: externalDisplay.bounds.y + 50,
    // ... 其他配置
  })
  win.loadURL('https://www.qq.com')
}
```

注意：显示器对象的 internal 属性用来说明是否是主显示器，但是目前无论是主显示器还是外接显示器，其值均为 false，所以判断是否是主显示器最好使用 `display.bounds` 判断。

### 4.2 自助机

自助机的操作系统一般为 Win、Linux、Android，如果是 PC 系统，则可以使用 Electron 开发，但是这类应用一般有以下特点：

- 大部分不允许用户退出
- 大部分支持触屏

在 `new BrowserWindow()` 时，Electron 为自助机提供了专用参数 kiosk。若该参数为 true，则窗口自动处于全屏状态，操作系统任务栏、窗口的默认标题都不再显示，窗口的高度与宽度设置也会失效。

触屏时，一般需要隐藏鼠标：

```css
body {
  cursor: none;
}
```

或者直接将鼠标锁定在可视区：

```js
document.body.requestPointerLock()
```

自助机的软键盘需要子进程配合：

```js
const { exec } = require('child_process')

exec('osk.exe')
```

## 五 硬件信息

### 5.1 获取硬件信息

```js
// 获取系统内存信息
let memoryUseage = process.getSystemMemoryInfo()

// 获取 CPU 信息
let cpuUseage = process.getCPUUseage()
```

更多的硬件信息，推荐使用第三方库<https://github.com/sebhildebrandt/systeminformation>

### 5.2 商业应用的授权问题

基础的授权方式为：获取用户设备的专有硬件信息（串号，全球唯一），将该信息、用户信息、付费信息绑定在一起，保存在服务端。

该授权方式有缺陷：必须联网才能验证，离线应用不可取；仍然可以通过代理请求破解验证。

针对离线应用，在第一次启动时，可以通过安全算法在本地生成一个与硬件匹配的激活码发送给用户，由用户保存，以后每次启动使用该激活码授权。该算法如果被破解，则可以无限分发激活码！
