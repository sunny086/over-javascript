# 08-鉴权 -2-Session

## 一 Session 简介

cookie 的数据有严重的安全问题，很容易被篡改、伪造，比如在上一节案例中，开发者直接发一个请求中带有 isLogin 的字段，服务端就判断其为登录状态了而且 Cookie 对敏感数据的保护是无效的。

Session 是专门为了解决 Cookie 数据安全性而提出来的技术，其实现基于 Cookie。

一般 Session 的实现步骤：

- 用户第一次访问服务器，服务器创建 session 对象，生成一个类似 key，value 的对象，然后将 key 返回给浏览器，以 cookie 的形式保存该 key。
- 用户再次访问时，携带了该 key，会得到对应的值。

## 二 Session 的实现

### 2.1 第一种 基于 Cookue 实现用户和数据的映射

可以将 Session 的口令存储在 Cookie 中，一旦口令被篡改，就会丢失映射关系，无法修改服务端数据了，而且 Session 有效期很短，通常为 20 分钟，也就是说，20 分钟内客户端与服务端之间没有交互产生，服务端就会将该 Cookie 中的口令数据删除。

```js
var sessions = {}
var key = 'session_id'
var EXPIRES = 20 * 60 * 1000

function generate() {
  var session = {}

  session.id = new Date().getTime() + Math.random()
  session.cookie = {
    expire: new Date().getTime() + EXPIRES,
  }

  sessions[session.id] = session

  return session
}
```

每个请求到来时，检查 Cookie 中的口令与服务端的数据，如果过期，就重新生成：

```js
function (req, res) {
    var id = req.cookies[key];
    if (!id) {
        req.session = generate();
    } else {
        var session = sessions[id];
        if (session) {
            if (session.cookie.expire > (new Date()).getTime()) {
                // 更新超时时间
                session.cookie.expire = (new Date()).getTime() + EXPIRES;
                req.session = session;
            } else {
                // 超时了，删除旧的数据，并重新生成
                delete sessions[id];
                req.session = generate();
            }
        } else {
            // 如果 session 过期或口令不对，重新生成 session
            req.session = generate();
        }
    }
    handle(req, res);
}
```

此时还需要在响应给客户端时设置新的值，以便下次请求时能够对应服务端的数据，这里重新实现 writeHead() 方法，在内部注入设置 Cookie 的逻辑：

```js
var writeHead = res.writeHead
res.writeHead = function () {
  var cookies = res.getHeader('Set-Cookie')
  var session = serialize('Set-Cookie', req.session.id)
  cookies = Array.isArray(cookies)
    ? cookies.concat(session)
    : [cookies, session]
  res.setHeader('Set-Cookie', cookies)
  return writeHead.apply(this, arguments)
}
```

业务逻辑使用 session：

```js
function handle(req, res) {
  if (!req.session.isLogin) {
    res.session.isLogin = true
    res.writeHead(200)
    res.end('欢迎登陆')
  } else {
    res.writeHead(200)
    res.end('请先登录')
  }
}
```

该方案依赖了 Cookie 的实现，也是大多 Web 系统中 Session 的实现方案，如果客户端禁止 Cookie，则本方案将会失效。

### 2.2 第二种 通过查询字符串来实现浏览器和服务端数据对应

该方案原理是检查请求的查询字符串，如果没有值，则会先生成新的带值的 URL，禁用 cookie 时可以采用该方案：

```js
function getURL(_url, key, value) {
  var obj = url.parse(_url, true)
  obj.query[key] = value
  return url.format(obj)
}
```

然后形成跳转，让客户端重新发起请求：

```js
function (req, res) {

    var redirect = function (url) {
        res.setHeader('Location', url);
        res.writeHead(302);
        res.end();
    };

    var id = req.query[key];

    if (!id) {
        var session = generate();
        redirect(getURL(req.url, key, session.id));
    } else {
        var session = sessions[id];
        if (session) {
            if (session.cookie.expire > (new Date()).getTime()) {
                // 更新超时时间
                session.cookie.expire = (new Date()).getTime() + EXPIRES;
                req.session = session;
                handle(req, res);
            } else {
                // 超时了，删除旧的数据，并重新生成
                delete sessions[id];
                var session = generate();
                redirect(getURL(req.url, key, session.id));
            }
        } else {
            // 如果 session 过期或口令不对，重新生成 session
            var session = generate();
            redirect(getURL(req.url, key, session.id));
        }
    }
}
```

用户访问`http://localhost/pathname`时，如果服务端发现查询字符串中没有 session_id 参数，就会将用户调转到 `http://localhost/pathname?session_id=12344567` 类似这样的地址，如果浏览器收到 302 状态码和 Location 报头，就会重新发起新的请求：

```txt
< HTTP/1.1 302 Moved Temporarily
< Location: /pathname?session_id=12344567
```

这样，新的请求到来时就能通过 Session 的检查。虽然该方案可以应对 Cookie 被禁用的情况，但是只要将地址栏中的地址发给另外一个人，他就会拥有和你相同的身份，风险更大！

## 三 集中式 Session 管理

在上述案例中 Session 都是存储在一个 Node 进程的变量中的。这会引起两个问题：

- 状态过多，如登录用户数目极大，会突破 Node 进程的内存限制，引起频繁 GC 扫描，造成性能问题
- Node 多进程中不共享内存，Session 就会出现错乱
- 在负载均衡状态下，多个服务器共同协作，用户的请求可能被不同的服务器执行，这时候其中一个服务器保存了 session，那么用户下次的请求在别的服务器上，将如何获取？

通常 Session 不会被考虑直接存储在业务进程中，一般将 session 保存在缓存服务器中，如 redis、memcache。

## 四 Session 安全

Session 的口令仍然是存储在 Cookie 中的，同样存在口令盗用的情况。通常可以对口令进行私钥加密签名，提升伪造成本。在服务端，只需要在响应数据时将口令和签名进行对比，如果签名非法，则服务端的数据立即过期即可：

```js
// 将值通过私钥签名，由 . 分割原值和签名
var sign = function (val, secret) {
  return (
    val +
    '.' +
    crypto
      .createHmac('sha256', secret)
      .update(val)
      .digest('base64')
      .replace(/\=+$/, '')
  )
}
```

在响应时，设置 session 值到 Cookie 中或者跳转 URL 中：

```js
var val = sign(req.sessionID, secret)
res.setHeader('Set-Cookie', cookie.serialize(key, val))
```

接收请求时，检查签名，对比用户提交的值：

```js
// 取出口令部分进行签名，对比用户提交的值
var unsign = function (val, secret) {
  var str = val.slice(0, val.lastIndexOf('.'))
  return sign(str, secret) == val ? str : false
}
```

这样一来，即使攻击者知道口令中 . 号的前的值是服务端 Session 的 ID 的值，只要不知道私钥的值，就无法伪造签名信息。

## 五 Session 在框架中的集成

### 5.1 Express 中使用 Session

```JavaScript
const express = require('express');
const session = require('express-session');

let app = express();

app.use(session({
//任意一个字符串，作为session的签名
    secret: 'sss',
    name: 'session_id', //session在本地cookie的名字，可以不设置
    //强制保存session，即使它没有变化，默认为true，建议为false
resave: false,
//强制将为初始化的session存储，默认是true
saveUninitialized: true,
cookie: {
   //不设置，那么关闭浏览器就过期，
//设置了即使在浏览页面，50000内没有操作也会过期
        maxAge: 50000
    },
    // secure https下才能访问cookie
//每次请求时强行设置cookue，将重置cookuie过期时间，默认false
    rolling: true
}));

app.get('/',function (req,res) {
    console.log(req.session.info);
    res.send('index');
});
app.get('/set',function (req,res) {
    req.session.info = 'lisi';
    res.send('set..');
});

app.listen(3000);
```

没有设置 maxAge，那么 session 在浏览器关闭时候被销毁，但是有时候用户即使仍在访问，我们也需要主动销毁 session，比如用户不关闭浏览器切换账户登录。
销毁方法一：req.session.cookie.maxAge = 0;
销毁方法二：req.session.destroy(function(err){})
