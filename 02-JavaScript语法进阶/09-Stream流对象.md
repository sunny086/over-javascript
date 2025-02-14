# 09-Stream 流对象

## 一 理解流

Streams API 诞生原因：Web 应用如何消费有序的小信息块，而不是大块信息，比如：

```txt
大块数据可能不会一次性都可用。比如网络请求中，网络负载是以连续信息包形式交付的，而流式处理可以让应用在数据一到达就能使用，而不必等到所有数据都加载完毕。

大块数据可能需要分小部分处理。视频处理、数据压缩、图像编码和 JSON 解析都是可以分成小部分进行处理，而不必等到所有数据都在内存中时再处理的例子。
```

流，类似于管道中液体的流动，所以 JS 也借用了管道这个概念，不过流是为映射低级 I/O 原语而设计的，主要解决的问题是：网络请求处理、磁盘读写处理。

流主要有三种：

```txt
可读流：可以通过某个公共接口读取数据块的流。数据在内部从底层源进入流，然后由消费者（consumer）进行处理。

可写流：可以通过某个公共接口写入数据块的流。生产者（producer）将数据写入流，数据在内部传入底层数据槽（sink）。

转换流：由两种流组成，可写流用于接收数据（可写端），可读流用于输出数据（可读端）。这两个流之间是转换程序（transformer），可以根据需要检查和修改流内容。
```

流的基本单位是块（chunk）。块可是任意数据类型，但通常是定型数组。每个块都是离散的流片段，可以作为一个整体来处理。更重要的是，块不是固定大小的，也不一定按固定间隔到达。在理想的流当中，块的大小通常近似相同，到达间隔也近似相等。不过好的流实现需要考虑边界情况。

有时候，由于数据进出速率不同，可能会出现不匹配的情况。为此流平衡可能出现如下三种情形：

```js
流出口处理数据的速度比入口提供数据的速度快。流出口经常空闲（可能意味着流入口效率较低），但只会浪费一点内存或计算资源，因此这种流的不平衡是可以接受的。

流入和流出均衡。这是理想状态。

流入口提供数据的速度比出口处理数据的速度快。这种流不平衡是固有的问题。此时一定会在某个地方出现数据积压，流必须相应做出处理。
```

流不平衡是常见问题，但流也提供了解决这个问题的工具。所有流都会为已进入流但尚未离开流的块提供一个内部队列。对于均衡流，这个内部队列中会有零个或少量排队的块，因为流出口块出列的速度与流入口块入列的速度近似相等。这种流的内部队列所占用的内存相对比较小。

如果块入列速度快于出列速度，则内部队列会不断增大。流不能允许其内部队列无限增大，因此它会使用反压（backpressure）通知流入口停止发送数据，直到队列大小降到某个既定的阈值之下。这个阈值由排列策略决定，这个策略定义了内部队列可以占用的最大内存，即高水位线（high water mark）。

## 二 可读流

### 2.0 可读流概念

可读流是对底层数据源的封装。底层数据源可以将数据填充到流中，允许消费者通过流的公共接口读取数据。

### 2.1 ReadableStreamDefaultController

`
下面的生成器每 1000 毫秒就会生成一个递增的整数：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
```

这个生成器的值可以通过可读流的控制器传入可读流。访问这个控制器最简单的方式就是创建 ReadableStream 的一个实例，并在这个构造函数的 underlyingSource 参数（第一个参数）中定义 start() 方法，然后在这个方法中使用作为参数传入的 controller。默认情况下，这个控制器参数是 ReadableStreamDefaultController 的一个实例：

```js
const readableStream = new ReadableStream({
  start(controller) {
    console.log(controller) // ReadableStreamDefaultController {}
  },
})
```

调用控制器的 enqueue() 方法可以把值传入控制器。所有值都传完之后，调用 close() 关闭流：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
const readableStream = new ReadableStream({
  async start(controller) {
    for await (let chunk of ints()) {
      controller.enqueue(chunk)
    }
    controller.close()
  },
})
```

### 2.2 ReadableStreamDefaultReader

前面的例子把 5 个值加入了流的队列，但没有把它们从队列中读出来。为此，需要一个 ReadableStreamDefaultReader 的实例，该实例可以通过流的 getReader() 方法获取。调用这个方法会获得流的锁，保证只有这个读取器可以从流中读取值：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
const readableStream = new ReadableStream({
  async start(controller) {
    for await (let chunk of ints()) {
      controller.enqueue(chunk)
    }
    controller.close()
  },
})

console.log(readableStream.locked) // false
const readableStreamDefaultReader = readableStream.getReader()
console.log(readableStream.locked) // true
```

消费者使用这个读取器实例的 read() 方法可以读出值：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}

const readableStream = new ReadableStream({
  async start(controller) {
    for await (let chunk of ints()) {
      controller.enqueue(chunk)
    }
    controller.close()
  },
})

console.log(readableStream.locked) // false
const readableStreamDefaultReader = readableStream.getReader()
console.log(readableStream.locked) // true

// 消费者
;(async function () {
  while (true) {
    const { done, value } = await readableStreamDefaultReader.read()
    if (done) {
      break
    } else {
      console.log(value)
    }
  }
})()
// 0
// 1
// 2
// 3
// 4
```

## 三 可写流

### 3.0 可写流概念

可写流是底层数据槽的封装。底层数据槽处理通过流的公共接口写入的数据。

### 3.1 创建 WritableStream

下面的生成器每 1000 毫秒就会生成一个递增的整数：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
```

这些值通过可写流的公共接口可以写入流。在传给 WritableStream 构造函数的 underlyingSink 参数中，通过实现 write() 方法可以获得写入的数据：

```js
const readableStream = new ReadableStream({
  write(value) {
    console.log(value)
  },
})
```

### 3.2 WritableStreamDefaultWriter

要把获得的数据写入流，可以通过流的 getWriter() 方法获取 WritableStreamDefaultWriter 的实例。这样会获得流的锁，确保只有一个写入器可以向流中写入数据：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}

const writableStream = new WritableStream({
  write(value) {
    console.log(value)
  },
})

console.log(writableStream.locked) // false
const writableStreamDefaultWriter = writableStream.getWriter()
console.log(writableStream.locked) // true
```

在向流中写入数据前，生产者必须确保写入器可以接收值。writableStreamDefaultWriter.ready 返回一个期约，此期约会在能够向流中写入数据时解决。然后，就可以把值传给 writableStreamDefaultWriter.write() 方法。写入数据之后，调用 writableStreamDefaultWriter.close() 将流关闭：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}

const writableStream = new WritableStream({
  write(value) {
    console.log(value)
  },
})

console.log(writableStream.locked) // false
const writableStreamDefaultWriter = writableStream.getWriter()
console.log(writableStream.locked) // true

// 生产者
;(async function () {
  for await (let chunk of ints()) {
    await writableStreamDefaultWriter.ready
    writableStreamDefaultWriter.write(chunk)
  }
  writableStreamDefaultWriter.close()
})()
```

## 四 转换流

转换流用于组合可读流和可写流。数据块在两个流之间的转换是通过 transform() 方法完成的。

下面的生成器每 1000 毫秒就会生成一个递增的整数：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
```

下面的代码创建了一个 TransformStream 的实例，通过 transform() 方法将每个值翻倍：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
const { writable, readable } = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk * 2)
  },
})
```

向转换流的组件流（可读流和可写流）传入数据和从中获取数据，与前面的方法相同：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}

const { writable, readable } = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk * 2)
  },
})

const readableStreamDefaultReader = readable.getReader()
const writableStreamDefaultWriter = writable.getWriter()

// 消费者
;(async function () {
  while (true) {
    const { done, value } = await readableStreamDefaultReader.read()
    if (done) {
      break
    } else {
      console.log(value)
    }
  }
})()

// 生产者
;(async function () {
  for await (let chunk of ints()) {
    await writableStreamDefaultWriter.ready
    writableStreamDefaultWriter.write(chunk)
  }
  writableStreamDefaultWriter.close()
})()
```

## 五 通过管道连接流

流可以通过管道连接成一串。最常见的用例是使用 pipeThrough() 方法把 ReadableStream 接入 TransformStream。从内部看，ReadableStream 先把自己的值传给 TransformStream 内部的 WritableStream，然后执行转换，接着转换后的值又在新的 ReadableStream 上出现。下面的例子将一个整数的 ReadableStream 传入 TransformStream，TransformStream 对每个值做加倍处理：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
const integerStream = new ReadableStream({
  async start(controller) {
    for await (let chunk of ints()) {
      controller.enqueue(chunk)
    }
    controller.close()
  },
})
const doublingStream = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk * 2)
  },
})
// 通过管道连接流
const pipedStream = integerStream.pipeThrough(doublingStream)
// 从连接流的输出获得读取器
const pipedStreamDefaultReader = pipedStream.getReader()
// 消费者
;(async function () {
  while (true) {
    const { done, value } = await pipedStreamDefaultReader.read()
    if (done) {
      break
    } else {
      console.log(value)
    }
  }
})()
// 0
// 2
// 4
// 6
// 8
```

使用 pipeTo() 方法也可以将 ReadableStream 连接到 WritableStream。整个过程与使用 pipeThrough() 类似：

```js
async function* ints() {
  // 每 1000 毫秒生成一个递增的整数
  for (let i = 0; i < 5; ++i) {
    yield await new Promise((resolve) => setTimeout(resolve, 1000, i))
  }
}
const integerStream = new ReadableStream({
  async start(controller) {
    for await (let chunk of ints()) {
      controller.enqueue(chunk)
    }
    controller.close()
  },
})
const writableStream = new WritableStream({
  write(value) {
    console.log(value)
  },
})
const pipedStream = integerStream.pipeTo(writableStream)
// 0
// 1
// 2
// 3
// 4
```

这里的管道连接操作隐式从 ReadableStream 获得了一个读取器，并把产生的值填充到 WritableStream。
