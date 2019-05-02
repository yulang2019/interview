# [HTTP 系列] 第 2 篇 —— HTTP 协议那些事

> 这里是《写给前端工程师的 HTTP 系列》，记得有位大佬曾经说过：“大厂前端面试对 HTTP 的要求比 CSS 还要高”，由此可见 HTTP 的重要程度不可小视。文章写作计划如下，视情况可能有一定的删减，本篇是该系列的第 2 篇 —— 《HTTP 协议那些事》。这篇文章会涉及到 HTTP 协议，cookie 和 session，HTTP 首部/方法/状态码，缓存机制。

更多文章可关注我的 [interview 系列](https://github.com/YanceyOfficial/interview)。

## 写作计划

- [从 TCP/UDP 到 DNS 解析](https://juejin.im/post/5cc5421e5188252e761e7e12)

- HTTP 协议那些事

- Web 服务器 (nginx/caddy)

- HTTPS (对称加密/非对称加密/SSL)

- JWT (我自己博客的后台用到了，所以专门要写一篇文章)

- SPDY / HTTP/2 / Websocket

- 网络攻击

- 跨域

- 缓存机制

- 浏览器原理

- 终章：从输入 url 到页面呈现发生了什么

## HTTP 协议

超文本传输协议（HyperText Transfer Protocol）是基于 TCP/IP 协议，用于分布式、协作式和超媒体信息系统的应用层协议。HTTP 是万维网的数据通信的基础，它是 `无状态` 协议，默认端口为 80。HTTP 在 TCP 的基础上，规定了 Request-Response 的模式，这个模式决定了通讯必定由浏览器首先发起。

抛去一些复杂的层面，浏览器开发者只需要一个 TCP 库就可以搞定浏览器的网络通讯部分。我们可以用 `telnet` 来做个实验。在 MAC 端直接使用 `brew install telnet` 即可。

首先连接到 `yanceyleo.com` 的主机。

```http
telnet yanceyleo.com 80
```

此时，三次握手完成，TCP 连接已经建立。输入下面内容，并 **双击回车**，就可以得到服务端响应的内容。下面的报文中，第一行的开头 `GET` 为请求访问服务器的类型，称为 `方法 (method)`；后面的 `/` 指明了请求访问的资源对象，也叫做请求 URI (request-URI)；最后为 HTTP 版本号，用来表示客户端使用的 HTTP 版本。第二行则是请求的主机名。

```http
GET / HTTP/1.1
Host: yanceyleo.com
```

![telnet 下的请求和响应](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190428-191318%402x.jpg)

### HTTP 是无连接、无状态协议

HTTP 是无状态 (stateless) 协议，它不会对请求和响应之间通信状态进行保存，也就是说 HTTP 协议不会对发送过的请求或响应做持久化处理。使用 HTTP 协议，每当有新的请求发送时，就会有对应的新响应产生。协议本身并不保留之前一切的请求或响应报文信息。这是为了更快地处理大量事务，确保协议的可伸缩性。

- 无连接

  每次连接只处理一个请求，服务端处理完客户端一次请求，等到客户端作出回应之后便断开连接。

- 无状态

  是指服务端对于客户端每次发送的请求都认为它是一个新的请求，上一次会话和下一次会话没有联系。

## cookie

### cookie 原理

何为 cookie 呢？我们在上面的文章中了解到 HTTP 是无状态的，但随着 Web 的不断发展，这种 **无状态** 的特性出现了弊端。当你登录到一家购物网站，在跳转到该站的其他页面时也应该继续保持登录状态，但是因为 HTTP 是无状态的，所以必须得在浏览器端存储一些信息来标识当前用户，因此 cookie 应运而生，它一种浏览器管理状态的文件。

![cookie 原理](https://yancey-assets.oss-cn-beijing.aliyuncs.com/07ecb36c4820a66de90013f303cac8c0.jpg)

浏览器第一次发出请求，服务器会将 cookie 放入到响应请求中，在浏览器第二次发请求的时候，会把 cookie 带过去，于是服务端就会辨别用户身份。注意：单个 cookie 保存的数据不能超过 4K，很多浏览器都限制一个站点最多保存 20 个 cookie。

### cookie 是不可跨域的

cookie 本身就是用来保存一些隐私性的字段，基于安全性的考量，必须要保证它是 **不可跨域的**。我们可以做个实验：先打开 `https://google.com`，然后在开发者工具中输入以下代码：

```js
document.cookie = 'hello=world;path=/;domain=.baidu.com';
document.cookie = 'world=hello;path=/;domain=.google.com';
```

打开 Application 选项卡，在侧边栏找到 Cookies，可以发现只有 domain 为 `.google.com` 的被成功添加。

![cookie 是不可跨域的](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190429-123624.jpg)

### cookie 的属性

我们通过一个登录的小例子来了解服务端设置 cookie。首先通过 `express --no-view mocker` 生成一个 express 的工程。关于 [express](https://expressjs.com/) 及其脚手架 [express application generator](https://expressjs.com/en/starter/generator.html) 这里不去多说。

\* 该 demo 的源码请访问 [express-cookies](https://github.com/YanceyOfficial/express-cookies)。

首先在 index.html 文件中输入以下代码，我们创建一个输入用户名和密码的界面，在点击按钮的时候，通过 fetch 将输入的值发送给后端。

```html
<fieldset>
  <legend>Login</legend>
  <input id="userName" type="text" placeholder="请输入用户名" />
  <input id="userPwd" type="password" placeholder="请输入密码" />
  <button id="loginBtn">登录</button>
</fieldset>

<p>登录状态: <span id="result"></span></p>
<script>
  const userName = document.getElementById('userName');
  const userPwd = document.getElementById('userPwd');
  const loginBtn = document.getElementById('loginBtn');
  const result = document.getElementById('result');

  loginBtn.addEventListener('click', function() {
    const data = {
      userName: userName.value,
      userPwd: userPwd.value
    };

    fetch('/login', {
      method: 'POST',
      // fetch 一定要手动设置 headers !!!
      headers: new Headers({
        'Content-Type': 'application/json'
      }),
      body: JSON.stringify(data)
    })
      .then(res => {
        return res.json();
      })
      .then(json => {
        result.innerHTML = json.msg;
      });
  });
</script>
```

当用户名和密码匹配时 (假设用户名和密码都是 `yancey`)，返回给客户端一个 cookie 以及登录成功的 json；否则返回登录失败的 json。下面是模拟服务端登录的接口。

```js
router.post('/login', (req, res, next) => {
  const body = req.body;
  if (body.userName === 'yancey' && body.userPwd === 'yancey') {
    // 设置 cookie
    res.cookie('yancey', 'success');
    res.json({
      success: true,
      msg: '登录成功'
    });
  } else {
    res.status(401).json({
      success: false,
      msg: '用户名或密码错误'
    });
  }
});
```

通过这个例子可以看到，在 express 中，setCookie 的方式为：第一个参数传递 `name`，第二个参数传递 `value`，**注意浏览器会将元字符和语义字符之外的字符进行转义**。打开 Chrome 的开发者工具，就可以看到该 cookie 被添加到浏览器上了。或者你在控制台输入 `document.cookie`，同样可以看到 cookie 字符串。

这只是一个设置 cookie 的简单例子，cookie 有 7 种属性可供使用，我们一一来了解。

![cookie 的属性](https://yancey-assets.oss-cn-beijing.aliyuncs.com/2340002414-566cde733b2cd_articlex.png)

#### domain

该属性给 cookie 设置 `域名`，默认为当前网站的域名，下面的例子将 domain 设为 **yanceyleo.com**，由于前端页面是 `127.0.0.1`，根据同源策略，该条 cookie 不会生效。

```js
res.cookie('domain', 'domian', { domain: 'yanceyleo.com' });
```

#### expires / maxAge

这两个属性都是设置 cookie 的 `过期时间`，不同的是，`expires` 接收一个 Date 格式的时间，而 `maxAge` 接收一个 `毫秒时间戳`。因为后者更加直观和简便，所以建议使用`maxAge`。

两个属性都可以传递一个 `负值` 或者 `0`，如果浏览器已存在同名 cookie，则会清除此 cookie，否则该条 cookie 不会被创建。

下面这个例子是创建一条 cookie，并将该 cookie 的过期时间设为一天后。

```js
res.cookie('expires', 'expires', {
  expires: new Date(Date.now() + 24 * 60 * 60 * 1000)
});
```

![设置过期时间](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190430-095753.jpg)

接着给该条 cookie 设置一个 “负数”，那么这条 cookie 就被清除了。

```js
res.cookie('expires', 'expires', {
  expires: new Date(Date.now() - 8 * 60 * 60 * 1000)
});
```

maxAge 的用法同理，它直接传递一个 `过期时间` 的毫秒数即可。下面的例子是将该条 cookie 的过期时间设为 7 天后。

```js
res.cookie('maxAge', 'maxAge', {
  maxAge: 7 * 24 * 60 * 60 * 1000
});
```

那么不设置过期时间的 cookie 会怎样呢？当你关闭该网站的时候，这些没有被设置过期时间的 cookie 就死翘翘了 (这种情况的 cookie 就好像是 session)。

#### httpOnly

当该属性设为 true 时，`document.cookie` 将无法获取该条 cookie，但服务端可以照常获得。该属性可以有效的避免跨站脚本攻击 (XSS)。关于网络安全方面的话题，后面会专门写一篇文章去讲。

```js
res.cookie('httpOnly', 'httpOnly', {
  // 只能被 web server 访问到，也就是说在浏览器输入 document.cookie 无法取到该条 cookie，目的是防止 xss
  httpOnly: true
});
```

#### path

该属性给 `指定的路径` 添加此 cookie，默认为 `/`。如下代码就是给 `users` 这个路由设置 cookie (即便在服务端该路径不存在也会被添加上)。

```js
res.cookie('path', 'path', {
  path: '/users'
});
```

![path 属性](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190430-145804.jpg)

#### secure

只有当连接是 HTTPS 协议，该 cookie 才会被添加。该属性默认为 fasle。因为我本地的 express 是 HTTP 协议，因此该条 cookie 不会生效。

```js
res.cookie('secure', 'secure', {
  secure: true
});
```

#### signed (防篡改签名)

该属性是给浏览器发送一个加密的 cookie，该属性默认为 false。在 express 中，我们可以使用 `cookie-parser` 插件来创建一个加密后的 cookie。服务端通过该 cookie 的内容和签名来检验它是否 `被篡改`

首先给 `cookieParser` 传入一个 secret。

```js
app.use(cookieParser('forcabarca'));
```

然后返回一个 sign 后的 cookie。

```js
res.cookie('signed', 'signed', {
  signed: true
});
```

![sign 属性](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190430-160532.jpg)

在 express 中，我们可以使用 `req.cookies` 来获得 `未加密` 的 cookie 对象，可以通过 `req.signedCookies` 来获得 `已加密` 的 cookie 对象。

```js
console.log(req.cookies); // { httpOnly: 'httpOnly' }
console.log(req.signedCookies); // { signed: 'signed' }
```

### `document.cookie` 字符串转对象的函数

关于 cookie 就说这么多，最后附赠一个 `document.cookie` 字符串转对象的函数，如果你有更好的实现方式，请在下面留言。

```js
const formatCookie = cookies => {
  const o = {};
  cookies
    .split(';')
    .forEach(value => (o[value.split('=')[0]] = value.split('=')[1]));
  return o;
};
```

## session

session 是服务端使用的一种记录客户端状态的机制，与 cookie 不同的是，session 保存在 服务端。当客户端初次发送请求时 (比如登录成功)，服务端会将用户信息以某种形式保存在服务端，当再次访问时只需从该 session 中找到该客户的状态即可。

因此，cookie 机制就是通过检查客户身上的 “通行证” 来确定客户身份，而 session 则是通过检查服务器上的 “客户明细表” 来确认客户身份。session 相当于程序在服务器上建立的一份客户档案，客户来访的时候只需要查询客户档案表就可以了。

因为 HTTP 是无状态的，所以单纯的 session 仍不能判断是否为到底是哪个用户。因此服务端仍要向客户端发送一个 `maxAge 为 -1` 的 cookie 来作为不同用户的唯一标识。

当然你也可以不使用 cookie，你可以通过重写 URL 地址的方式来实现。它的原理是将用户的 seesion id 写入到 URL 中，当浏览器解析新的 URL 时就可以定位到是哪位用户。

万变不离其宗，两种方式都是要保证用户信息以某种形式保存到客户端。更先进的 localStorage，sessionStorage，IndexedDB 也是同样的道理，这里不去细说。

## HTTP 报文

用于 HTTP 协议交互的信息被称为 HTTP 报文。客户端的报文叫做请求报文，服务端的报文叫做响应报文。HTTP 报文本身是有多行数据构成的字符串文本。

### 报文格式

![160c90cb78e72587.jpg](https://yancey-assets.oss-cn-beijing.aliyuncs.com/160c90cb78e72587.jpg)

上面这张图清晰地展示了请求报文和响应报文的格式，用文字描述大致如下。

> CR (Carriage Return，回车符：16 进制 0x0d)
>
> LF (Line Feed，换行符：16 进制 0x0a)

```xml
<!--请求报文-->
<method>空格<request-url>空格<version>
<headers>
空行 (CR + LF)
<entity-body>

<!--响应报文-->
<version>空格<status>空格<reason-phrase>
<headers>
空行 (CR + LF)
<entity-body>
```

### 压缩报文

HTTP 协议中有一种被称为 `内容编码` 的功能，可以有效的压缩报文的体积。内容编码指明应用在实体内容上的编码格式，并保持实体信息原样压缩。内容编码后的实体由客户端接收并负责解码。常见的内容编码有以下几种：

- compress (UNIX 系统的标准压缩)

- gzip (GNU zip, 可以说是目前网站中最常见的压缩方式)

- deflate (zlib)

- brotli (Google 出品，必属精品。比 gzip 的压缩率还要高 37%+，我的网站已使用 brotli，看下图)

- identity (不做压缩)

![压缩报文](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190501-212843.jpg)

### 分割发送的分块传输编码

从 HTTP 请求回来，就产生了流式的数据，后续的 DOM 树构建、CSS 计算、渲染、合成、绘制，都是尽可能地流式处理前一步的产出：即不需要等到上一步骤完全结束，就开始处理上一步的输出，这样我们在浏览网页时，才会看到逐步出现的页面。

本质上来说，在 HTTP 通信过程中，请求的编码实体资源尚未全部传输完成之前，浏览器无法显示请求页面。在传输大容量数据时，通过把数据分割成多块，能让浏览器逐步显示页面。这种把实体主体分块的功能称为分块传输编码 (Chunked Transfer Code)。

分块传输编码会将实体主体分为多个块，每个块都会使用十六进制来标记大小，而实体主体的最后一块会使用 `0 (CR+LF)` 来标记。使用分块传输编码的实体主体会由接收的客户端负责解码，恢复到编码前的实体主体。

## HTTP 报文首部

上面的章节我们说到了 HTTP 的报文，它由三部分组成，分别是：`报文首部`、`空行`、`报文主体`。

对于请求报文，它的首部由方法、URL、HTTP 版本、HTTP 首部字段等部分构成。

![Jietu20190501-224131@2x.jpg](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190501-224131%402x.jpg)

对于响应报文，它的首部分别由 HTTP 版本、状态码、HTTP 首部字段等部分构成。

![Jietu20190501-224131@2x.jpg](https://yancey-assets.oss-cn-beijing.aliyuncs.com/Jietu20190501-224131%402x.jpg)

### 首部字段类型

- **通用首部字段 (General Header Field)** 请求报文和响应报文两方都会使用的首部。

- **请求首部字段 (Request Header Field)** 从客户端向服务端发送请求报文时使用的首部。补充了请求的附加内容、客户端信息、响应内容相关优先级等信息。

- **响应首部字段 (Response Header Field)** 从服务端向客户端返回响应报文时使用的首部。补充了响应的附加内容，也会要求客户端附加额外的内容信息。

- **实体首部字段 (Entity Header Field)** 针对请求报文和响应报文的实体部分使用的首部。补充了资源内容更新时间等与实体有关的信息。

### 首部字段一览

通用首部字段：

| 首部字段名 | 说明 |
| --- | --- |
| Cache-Control |  |
| Connection |  |
|  |  |

## HTTP 方法

## HTTP 状态码

## 参考

[express 中 cookie 的使用和 cookie-parser 的解读](https://segmentfault.com/a/1190000004139342)

[谈谈 cookie](http://barryliu1995.studio/2017/10/11/谈谈cookie)

[把 cookie 聊清楚](https://juejin.im/post/59d1f59bf265da06700b0934)
