---
title: 从原生 HTTP 到 Express API： Node.js 后端入门路线与踩坑复盘
published: 2026-09-21
description: 从 Node.js 原生 http、fs、path 与 CommonJS 出发，逐步理解 npm、Express、Router、中间件和模块化 API；同时复盘请求未结束、参数混淆、导出失效、路由前缀重复与错误处理中间件顺序等常见问题。
image: ""
tags: [JavaScript, Node.js, Express, 后端开发]
category: Node.js 学习
draft: false
lang: zh-CN
---

## 糊点

这一阶段的学习路线看起来由许多零散知识组成：先用 `http.createServer()` 启动服务器，再认识 `req` 和 `res`，然后读取文件、学习模块化、安装 npm 包，最后进入 Express 的路由与中间件。真正把这些内容串起来之后，会发现它们始终围绕着同一个问题：**一个 HTTP 请求进入 Node.js 以后，究竟经过了什么，又是怎样得到响应的？**

起初可以把服务器理解成“监听一个端口，然后返回一段文字”。这个认识就好比中学时期的理科，虽然在一定体系内是正确的，但是并不成为客观世界的正确解释。一个稍微像样的服务至少还要回答这些问题：

- 请求使用了什么 HTTP 方法，请求的路径是什么？
- 查询参数、动态路径参数和请求体分别放在哪里？
- 哪段代码先执行，哪段代码可以终止请求？
- 静态文件如何安全地映射到磁盘文件？
- 多个功能怎样拆成模块，而不是全部堆在一个文件里？
- 业务出错、路由不存在、请求体格式错误时怎样响应？

沿着这些问题继续往下走，原生 `http` 模块负责的事情会逐渐被拆开：路由负责把请求交给正确的处理函数，中间件负责处理多个路由都需要的公共逻辑，Router 负责按业务拆分代码，错误处理中间件则负责统一收尾。在后续笔者大量地使用到了express，它把常见操作整理成了一套更易用的接口。

这篇文章是我的梦话。

---

## 一、第一次创建 Web 服务器

### 1. 最小的 Node.js HTTP 服务

Node.js 内置了 `http` 模块，不安装第三方包也能创建 Web 服务器：

```js
const http = require('http')

const server = http.createServer()

server.on('request', (req, res) => {
  console.log(`${req.method} ${req.url}`)

  res.statusCode = 200
  res.setHeader('Content-Type', 'text/plain; charset=utf-8')
  res.end('你好，Node.js')
})

server.listen(3000, '127.0.0.1', () => {
  console.log('Server running at http://127.0.0.1:3000')
})
```

运行 `node server.js` 后访问 `http://127.0.0.1:3000`，一次最基本的请求会经历以下过程：

1. `server.listen()` 让进程监听 3000 端口。
2. 浏览器或其他客户端建立连接并发送 HTTP 请求。
3. `server` 触发 `request` 事件，Node.js 调用注册的处理函数。
4. `req` 提供请求信息，`res` 用来构造响应。
5. `res.end()` 发送最后一段响应数据，并明确告诉 Node.js：这次响应结束了。

这里最容易被忽略的是第 5 步。服务器收到请求并打印日志，并不等于客户端已经拿到响应。如果处理函数只写：

```js
server.on('request', (req, res) => {
  console.log('Someone visited our web server')
})
```

浏览器通常会一直处于加载状态，因为服务端既没有结束响应，也没有关闭连接。这类代码可以通过 `node --check`，启动时也不一定报错，却在真正发出请求后暴露问题。因此，**语法正确不等于运行逻辑正确**。

### 2. `req` 和 `res` 分别负责什么

`req` 是可读的请求对象，`res` 是可写的响应对象。入门阶段可以先记住这些常用成员：

| 对象 | 成员 | 作用 |
|------|------|------|
| `req` | `method` | HTTP 方法，例如 `GET`、`POST` |
| `req` | `url` | 路径和查询字符串，例如 `/users?page=2` |
| `req` | `headers` | 客户端发送的请求头 |
| `res` | `statusCode` | 设置 HTTP 状态码 |
| `res` | `setHeader()` | 设置响应头 |
| `res` | `write()` | 写入一段响应内容，可以调用多次 |
| `res` | `end()` | 写入可选的最后一段内容并结束响应 |

`req.url` 并不只是“页面名称”。当客户端请求 `/search?keyword=node&page=1` 时，它的值通常包含路径和查询字符串。原生 Node.js 中可以使用 WHATWG `URL` API 拆分：

```js
const requestUrl = new URL(req.url, 'http://127.0.0.1')

console.log(requestUrl.pathname)                  // /search
console.log(requestUrl.searchParams.get('keyword')) // node
console.log(requestUrl.searchParams.get('page'))    // 1
```

第二个参数是解析相对 URL 所需的基准地址，并不表示程序又发起了一次网络请求。

### 3. 响应头、编码与状态码

如果响应包含中文，应明确声明字符编码：

```js
res.setHeader('Content-Type', 'text/html; charset=utf-8')
res.end('<h1>首页</h1>')
```

`Content-Type` 同时告诉客户端两件事：响应内容是什么格式，以及文本使用什么字符集。返回 JSON 时则应使用：

```js
res.statusCode = 200
res.setHeader('Content-Type', 'application/json; charset=utf-8')
res.end(JSON.stringify({ message: '请求成功' }))
```

状态码不只是装饰。常见状态码可以先掌握以下几种：

| 状态码 | 常见含义 |
|--------|----------|
| `200 OK` | 请求成功 |
| `201 Created` | 成功创建资源 |
| `400 Bad Request` | 请求格式或参数不合法 |
| `404 Not Found` | 没有匹配的资源或路由 |
| `405 Method Not Allowed` | 路径存在，但不支持该 HTTP 方法 |
| `500 Internal Server Error` | 服务端出现未预期错误 |

### 踩坑复盘

- **只打印日志，没有 `res.end()`**：服务端看起来“收到请求了”，客户端却一直等待。
- **直接使用 80 端口**：80 是常用系统端口，更容易遇到权限限制或端口占用；练习项目统一使用 3000 会更省事。
- **把 `req.url` 当作纯路径**：带查询参数时，`req.url === '/users'` 不会匹配 `/users?page=1`。
- **没有声明 UTF-8**：中文可能被错误解码。编码属于响应协议的一部分，不是编辑器显示设置。

服务器必须针对每个请求明确地构造并结束响应。

---

## 二、从手写路由到静态资源服务器

### 1. 根据 URL 返回不同内容

在原生 `http` 中，最直接的路由就是条件判断：

```js
const http = require('http')

const server = http.createServer((req, res) => {
  const { pathname } = new URL(req.url, 'http://127.0.0.1')

  res.setHeader('Content-Type', 'text/html; charset=utf-8')

  if (req.method !== 'GET') {
    res.statusCode = 405
    res.setHeader('Allow', 'GET')
    res.end('<h1>405 Method Not Allowed</h1>')
    return
  }

  if (pathname === '/' || pathname === '/index.html') {
    res.statusCode = 200
    res.end('<h1>首页</h1>')
    return
  }

  if (pathname === '/about.html') {
    res.statusCode = 200
    res.end('<h1>关于页面</h1>')
    return
  }

  res.statusCode = 404
  res.end('<h1>404 Not Found</h1>')
})

server.listen(3000, () => {
  console.log('Server running at http://127.0.0.1:3000')
})
```

这里的 `return` 不是 HTTP 协议要求，而是为了结束当前 JavaScript 函数，避免发送响应后代码继续向下运行。随着路由增多，大量 `if...else` 会变得难以维护，这正是后面引入 Express 路由的原因之一。

### 2. `fs`、`path` 与错误优先回调

读取文件时常用 `fs` 和 `path`：

```js
const fs = require('fs')
const path = require('path')

const filePath = path.join(__dirname, 'exp.txt')

fs.readFile(filePath, 'utf8', (err, data) => {
  if (err) {
    console.error(`读取失败：${err.message}`)
    return
  }

  console.log(data)
})
```

Node.js 传统回调大多遵循“错误优先”约定：第一个参数是 `err`，成功时通常为 `null`；后面的参数才是结果。原笔记中有一处回调形参写成 `err`，错误分支却访问 `error.message`。这不会造成语法错误，但读取失败时会抛出 `ReferenceError: error is not defined`，反而遮住真正的文件错误。

`path.join()` 或 `path.resolve()` 比手写斜杠更可靠，因为 Windows 和类 Unix 系统使用的路径分隔符不同。`__dirname` 表示当前模块所在目录，不受启动命令所在目录影响；相对路径则通常根据 `process.cwd()` 解析，两者不能混为一谈。

### 3. 一个更完整的静态文件服务器

在最初实现 Clock 页面时，核心思路大致是：将文件的实际存放位置映射为资源的请求 URL，服务器充当“字符串的搬运工”。客户端要 HTML，就读出 HTML；浏览器继续请求 CSS 和 JavaScript，就把对应文件的内容再搬进响应里。这个理解是静态服务器最直观的工作方式，但服务器不只是搬运内容，还要决定客户端能搬走什么、内容应该按什么类型解析，以及文件不存在时如何回答。

静态服务器不能简单地把 `req.url` 拼接到磁盘目录后面。客户端可以构造 `../` 尝试访问公开目录之外的文件，这被称为目录穿越。下面的示例加入了路径解码、范围校验、MIME 类型、404 和 500 响应：

```js
const http = require('http')
const fs = require('fs')
const path = require('path')

const publicRoot = path.resolve(__dirname, 'clock')

const mimeTypes = {
  '.html': 'text/html; charset=utf-8',
  '.css': 'text/css; charset=utf-8',
  '.js': 'text/javascript; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.svg': 'image/svg+xml'
}

const server = http.createServer((req, res) => {
  if (req.method !== 'GET' && req.method !== 'HEAD') {
    res.writeHead(405, { Allow: 'GET, HEAD' })
    res.end('Method Not Allowed')
    return
  }

  let pathname

  try {
    pathname = decodeURIComponent(
      new URL(req.url, 'http://127.0.0.1').pathname
    )
  } catch {
    res.writeHead(400, { 'Content-Type': 'text/plain; charset=utf-8' })
    res.end('Bad Request')
    return
  }

  const resourcePath = pathname === '/' ? '/index.html' : pathname
  const filePath = path.resolve(publicRoot, `.${resourcePath}`)
  const insidePublicRoot =
    filePath === publicRoot || filePath.startsWith(`${publicRoot}${path.sep}`)

  if (!insidePublicRoot) {
    res.writeHead(403, { 'Content-Type': 'text/plain; charset=utf-8' })
    res.end('Forbidden')
    return
  }

  fs.readFile(filePath, (err, data) => {
    if (err) {
      const statusCode = err.code === 'ENOENT' ? 404 : 500
      const message = statusCode === 404 ? 'Not Found' : 'Internal Server Error'

      res.writeHead(statusCode, {
        'Content-Type': 'text/plain; charset=utf-8'
      })
      res.end(message)
      return
    }

    const extension = path.extname(filePath).toLowerCase()
    const contentType = mimeTypes[extension] || 'application/octet-stream'

    res.writeHead(200, { 'Content-Type': contentType })
    res.end(req.method === 'HEAD' ? undefined : data)
  })
})

server.listen(3000, () => {
  console.log('Static server running at http://127.0.0.1:3000')
})
```

这段代码比最初“服务器是字符串搬运工”的理解多了几个边界条件：服务器不仅要搬运内容，还要决定客户端能搬走什么、内容是什么类型、文件不存在时如何回答。

### 踩坑复盘

- **变量名写错**：`err` 与 `error` 不一致属于运行时错误，只有走进错误分支才会出现。
- **直接拼接 URL 与文件路径**：URL 使用 `/`，文件系统路径由操作系统决定；未经校验还会产生目录穿越风险。
- **读取失败直接抛错**：一个文件不存在不应该让整个服务崩溃，应转换为合适的 HTTP 响应。
- **多个练习服务同时监听同一端口**：后启动的进程会遇到 `EADDRINUSE`。可以停止旧进程或为不同示例使用不同端口。
- **只给所有文件设置 `text/html`**：CSS、JavaScript、图片都有各自的 MIME 类型，类型错误会影响浏览器解析。

---

## 三、CommonJS：把代码拆成模块

当所有代码都写在一个文件里时，变量容易冲突，功能也很难复用，而模块作用域可以防止全局变量污染。一个文件里的局部变量默认不会直接跑到另一个文件中；如果确实想把某项能力交给外部使用，可以通过 `module.exports` 主动建立接口。Node.js 的 CommonJS 模块系统正是沿着这个边界组织代码。

### 1. `require()` 实际拿到了什么

假设有一个 `userService.js`：

```js
const users = [
  { id: 1, name: 'Ada' },
  { id: 2, name: 'Linus' }
]

function findUserById(id) {
  return users.find((user) => user.id === id)
}

module.exports = {
  users,
  findUserById
}
```

另一个模块可以导入它：

```js
const { users, findUserById } = require('./userService')

console.log(users)
console.log(findUserById(1))
```

`require('./userService')` 得到的是该模块最终的 `module.exports`。首次加载时，模块顶层代码会执行；加载完成后，结果会进入缓存。同一进程再次加载解析到同一文件的模块时，通常直接返回缓存中的导出对象，而不是从头执行一遍。

模块缓存能避免重复初始化，但也意味着模块顶层的可变对象可能成为进程内共享状态。内存用户数组、数据库连接实例和计数器都要意识到这一点。

### 2. `exports` 为什么有时会失效

Node.js 初始化模块时，可以把关系简化理解为：

```js
exports === module.exports // true
```

理解这层关系时，可以考虑：如果两个变量指向同一个对象，通过其中一个变量修改对象，另一个变量看到的也会是修改后的结果。因此，`exports.username = 'zc'` 会反映到 `module.exports` 上。入门时常把它说成“变量保存了对象的地址值”，用来想象指向关系没有问题；在更严谨的表述中，它保存的是对象引用，并不代表 JavaScript 暴露了一个可以直接操作的物理内存地址。

因此，给 `exports` 增加属性是有效的：

```js
exports.username = 'zc'
exports.sayHello = function sayHello() {
  console.log('Hello')
}
```

但直接让 `exports` 指向一个新对象，只改变了局部变量的指向：

```js
exports = {
  username: 'zc'
}
```

此时 Node.js 最终返回的仍然是原来的 `module.exports`。如果想整体替换导出值，应明确写：

```js
module.exports = {
  username: 'zc'
}
```

为了减少“两个名字到底指向谁”的心智负担，一个模块中最好选择一种主要写法；需要整体导出对象时，直接使用 `module.exports` 最清楚。如果你学过cpp和指针，这个就很好理解了。

### 3. CommonJS 与 ES Modules 的简单对照

| CommonJS | ES Modules |
|----------|------------|
| `require()` | `import` |
| `module.exports` | `export` / `export default` |
| Node.js 传统模块格式 | JavaScript 标准模块格式 |
| 本项目当前使用 | 通常需要 `.mjs` 或配置 `"type": "module"` |

两套模块系统都能组织代码，但语法、文件解析和部分运行行为不同。本文保持与现有项目一致，所有可运行示例统一使用 CommonJS.



模块化主要的贡献在于，每个文件开始拥有明确职责和对外接口，这对后续的 Router 拆分是很有帮助的。

---

## 四、npm、包与可复现的依赖

Node.js 核心模块随运行时提供，Express 这类第三方功能则以 npm 包的形式安装。npm 不只是下载工具，它还负责记录项目依赖与可执行脚本。

### 1. `package.json` 与 `package-lock.json`

一个常见的 `package.json` 可以写成：

```json
{
  "name": "node-learning-api",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  },
  "dependencies": {
    "express": "^5.2.1"
  },
  "devDependencies": {
    "nodemon": "^3.1.14"
  }
}
```

- `dependencies` 是程序正常运行需要的依赖，例如 Express。
- `devDependencies` 是开发、测试或构建阶段使用的工具，例如 nodemon。
- `scripts` 为常用命令提供统一入口，运行时使用 `npm run dev` 或 `npm start`。
- `package-lock.json` 锁定实际解析到的依赖树，让不同机器安装出尽可能一致的版本。

一般应把 `package.json` 和 `package-lock.json` 一起提交，而不提交体积庞大的 `node_modules`。拿到项目后运行 `npm install`，npm 根据清单恢复依赖。

### 2. 版本号中的 `^` 是什么意思

语义化版本通常写作 `主版本.次版本.修订版本`。以 `^5.2.1` 为例，在主版本非零时，它通常允许安装兼容的 `5.x.x` 新版本，但不会自动跨到 `6.0.0`。因此它是一个版本范围，不是“永远固定在 5.2.1”。

范围用于接收兼容更新，锁文件则记录本次实际安装结果。两者解决的是不同问题：前者表达允许范围，后者提高安装的可复现性。

### 3. 当前项目里的两个包

当前学习项目使用 Express 5.2.1 和 nodemon 3.1.14：

- **Express** 提供路由、中间件、响应辅助方法和静态资源服务等能力。
- **nodemon** 监控源文件变化并重启 Node.js 进程，只改善开发体验，不参与线上请求处理。


---

## 五、Express：你们在这里封装得很开心嘛

*私密马赛，私密马赛*

### 1. 最小 Express 服务

原生 `http` 需要自行判断方法、路径、响应格式。Express 把这些模式封装为更直观的 API：

```js
const express = require('express')

const app = express()

app.get('/', (req, res) => {
  res.send('Home page')
})

app.get('/health', (req, res) => {
  res.status(200).json({
    code: 0,
    message: '服务正常',
    data: null
  })
})

app.listen(3000, () => {
  console.log('Express server running at http://127.0.0.1:3000')
})
```

`app.get(path, handler)` 同时描述了请求方法、路径和处理函数。`res.send()` 会根据数据设置合适的响应类型并结束响应；`res.json()` 明确返回 JSON；`res.status()` 用链式方式设置状态码。

一次请求只能完成一次响应。下面的写法会尝试发送两次：

```js
app.get('/wrong', (req, res) => {
  res.send('first response')
  res.send('second response')
})
```

第二次发送通常会触发 `ERR_HTTP_HEADERS_SENT`。发送响应后如果当前函数还有其他分支，应使用 `return res.status(...).json(...)` 结束函数，降低误执行后续代码的风险。

### 2. `query`、`params` 和 `body`

这三类输入来自请求的不同位置：

| 输入 | 示例 | Express 中的读取方式 | 常见用途 |
|------|------|----------------------|----------|
| 查询参数 | `/users?keyword=ada` | `req.query.keyword` | 筛选、分页、排序 |
| 路径参数 | `/users/42` | `req.params.id` | 标识一个具体资源 |
| 请求体 | POST JSON | `req.body` | 创建或修改资源的数据 |

示例：

```js
const express = require('express')

const app = express()

app.use(express.json())
app.use(express.urlencoded({ extended: false }))

app.get('/users', (req, res) => {
  res.json({ keyword: req.query.keyword ?? '' })
})

app.get('/users/:id', (req, res) => {
  res.json({ id: req.params.id })
})

app.post('/users', (req, res) => {
  res.status(201).json({ user: req.body })
})

app.listen(3000)
```

Express 不会凭空知道怎样解析任意请求体。`express.json()` 负责匹配并解析 JSON 请求体，`express.urlencoded()` 负责表单常用的 URL-encoded 格式。它们从 Express 4.16.0 起成为内置中间件，在当前 Express 5.2.1 中可以直接使用。

### 3. 静态资源

```js
app.use(express.static(path.join(__dirname, 'public')))
app.use('/assets', express.static(path.join(__dirname, 'assets')))
```

第一行会让 `public/index.html` 通过 `/index.html` 访问，目录名 `public` 不出现在 URL 中。第二行显式增加 `/assets` 前缀。注册多个静态目录时，Express 按中间件注册顺序查找；同名资源由先匹配成功的目录提供。

### 4. 推荐的应用顺序

Express 的行为高度依赖注册顺序。一个清晰的应用通常按以下顺序组织：

```text
创建 app
  ↓
注册请求体解析、日志等全局中间件
  ↓
注册静态资源和业务 Router
  ↓
注册 404 处理
  ↓
注册错误处理中间件
  ↓
调用 app.listen()
```

把 `app.use()` 写在 `app.listen()` 后面时，由于这些语句通常在同一轮同步执行，路由不一定立刻失效；但这种组织方式模糊了“配置完成后再启动”的边界，也容易造成阅读者误判。统一把中间件和路由注册放在监听之前，更符合应用初始化过程。


---

## 六、路由与 Router 模块化

### 1. 路由是一种映射关系

Express 路由可以概括为：

```text
HTTP 方法 + 请求路径 → 一个或多个处理函数
```

例如：

```js
app.get('/users', listUsers)
app.post('/users', createUser)
```

路径相同而方法不同，代表不同操作。Express 会按照注册顺序检查处理层。一个处理函数如果已经发送响应，就应当结束；如果调用 `next()`，Express 会继续向后寻找下一个可执行的中间件或路由处理函数。因此，“匹配到路由后绝对不会继续匹配”并不准确，是否继续取决于处理函数做了什么。

### 2. 为什么需要 `express.Router()`

当用户、文章、订单都写在 `app.js` 中时，入口文件会迅速膨胀。Router 可以把一组相关路由拆成独立模块：

```js
// routes/users.js
const express = require('express')

const router = express.Router()

router.get('/', (req, res) => {
  res.json({ message: '用户列表' })
})

router.post('/', (req, res) => {
  res.status(201).json({ message: '创建用户' })
})

module.exports = router
```

入口文件只负责装配：

```js
// app.js
const express = require('express')
const usersRouter = require('./routes/users')

const app = express()

app.use(express.json())
app.use('/api/users', usersRouter)

app.listen(3000)
```


Router 内部的 `/` 与挂载前缀 `/api/users` 合并后，最终地址是 `/api/users`。如果 Router 内部又写成 `/api/users`，最终就会变成 `/api/users/api/users`。

### 3. `app` 与 `router` 的职责

为了区分 `app` 和 Router，可以整个应用想成一座商场：`app` 是商场的总入口，用户区、文章区和订单区则像不同楼层。请求先进入商场，再根据路径被带到某个楼层，之后由该 Router 负责区域内部的功能。但是楼层本身不能独立开门营业，Router 也不能自己监听端口；真正调用 `listen()`、接受网络连接的仍然是整个应用。

| 能力 | `app` | `router` |
|------|-------|----------|
| 注册路由和中间件 | 是 | 是 |
| 按业务拆分处理链 | 可以，但不宜全部堆在一起 | 核心用途 |
| 统一装配整个应用 | 是 | 否 |
| `listen()` 启动服务 | 是 | 否 |

Router 不是一个单独运行的服务器。它要通过 `app.use()` 挂载到应用的请求处理链中。


---

## 七、中间件：理解 Express 的请求处理链

### 1. 中间件的基本形式

中间件就是业务流程中的中间处理环节。请求从入口走向最终路由时，可以依次经过日志、请求体解析和鉴权；每一层完成自己的工作，再决定让请求继续向后走，还是直接给客户端响应。

普通中间件本质上是一个函数：

```js
function middleware(req, res, next) {
  // 读取或修改 req、res
  // 决定继续传递，还是直接发送响应
}
```

如果某一层只负责发送响应，它完全可以只使用 `req` 和 `res`；需要把控制权交给后续处理层时，才需要接收并调用 `next`。路由处理函数也同样可以使用第三个参数，所以“中间件一定有三个参数、路由一定只有两个参数”并不是它们的本质区别。

它有两种正常结束方式：

1. 调用 `next()`，把控制权交给后续处理层。
2. 发送响应，结束这次请求，不再调用 `next()`。

所以“中间件必须调用 `next()`”只说对了一半。更准确的规则是：**中间件必须调用 `next()` 或结束响应，不能什么都不做。**

### 2. 全局中间件与局部中间件

通过 `app.use()` 注册的无路径中间件通常对后续请求全局生效：

```js
app.use((req, res, next) => {
  req.startTime = process.hrtime.bigint()
  next()
})
```

局部中间件只参与指定路由：

```js
function requireApiKey(req, res, next) {
  if (req.get('x-api-key') !== 'study-key') {
    res.status(401).json({
      code: 401,
      message: '缺少有效的 API Key',
      data: null
    })
    return
  }

  next()
}

app.get('/private', requireApiKey, (req, res) => {
  res.json({ code: 0, message: '访问成功', data: null })
})
```

校验失败时，中间件已经发送 401 响应，因此不能继续调用 `next()`；校验成功才交给路由处理函数。



### 3. 多个中间件共享 `req` 和 `res`

同一次请求经过的多个中间件共享同一份 `req` 和 `res`。这意味着我可以在上游中间件里给请求对象补充 `startTime` 之类的数据，再让下游中间件或路由继续使用。它很适合记录请求级上下文，不过正式项目中应选择清晰且不会与框架字段冲突的属性名。下面的例子在上游记录时间，并在响应完成后计算耗时：

```js
app.use((req, res, next) => {
  const startedAt = process.hrtime.bigint()

  res.on('finish', () => {
    const elapsedNanoseconds = process.hrtime.bigint() - startedAt
    const elapsedMilliseconds = Number(elapsedNanoseconds) / 1e6

    console.log(
      `${req.method} ${req.originalUrl} ${res.statusCode} ${elapsedMilliseconds.toFixed(2)}ms`
    )
  })

  next()
})
```

监听响应的 `finish` 事件，比单纯在 `next()` 后打印更接近“响应已经完成”的时间点。因为 `next()` 并不是跳出当前函数，它会进入后续处理层；当后续处理完成后，当前函数中 `next()` 后面的同步代码仍可能继续执行。这种行为有点像嵌套调用：先向里走，再逐层返回。



### 4. 中间件的常见分类

| 分类 | 注册位置或形式 | 典型用途 |
|------|----------------|----------|
| 应用级 | `app.use()`、`app.get()` | 全局日志、应用路由 |
| 路由级 | `router.use()`、`router.get()` | 某一业务模块的处理链 |
| 内置 | `express.json()`、`express.static()` | 请求体解析、静态文件 |
| 第三方 | 通过 npm 安装后注册 | CORS、安全响应头、日志等 |
| 错误处理 | `(err, req, res, next)` | 统一处理进入错误链的异常 |

### 5. 错误处理中间件

错误处理中间件必须保留四个参数，即使暂时不用 `next`：

```js
app.use((err, req, res, next) => {
  console.error(err)

  if (res.headersSent) {
    next(err)
    return
  }

  res.status(500).json({
    code: 500,
    message: '服务器内部错误',
    data: null
  })
})
```

它通常注册在业务路由和 404 处理之后，这样前面产生的错误才能流入这里。普通同步路由中抛出的异常会被 Express 捕获。Express 5 还会把 `async` 处理函数返回的 Promise 拒绝自动传给错误处理流程：

```js
app.get('/async-error', async (req, res) => {
  await Promise.reject(new Error('异步操作失败'))
  res.send('不会执行到这里')
})
```

这项行为与早期 Express 4 项目中常见的手动包装方式不同。即便框架会转发错误，应用仍然需要在末尾注册错误处理中间件，才能得到统一响应。

### 踩坑复盘

- **忘记 `next()`**：中间件既不响应也不向后传递，请求会一直挂起。
- **调用 `next()` 后继续发送响应**：下游可能已经完成响应，回到上游后再次写入就会出错。
- **解析中间件放得太晚**：路由执行时 `req.body` 还没有被填充。
- **错误处理中间件少写一个参数**：Express 会把它当普通中间件，而不是错误处理器。
- **错误处理器放在业务路由前面**：之后路由抛出的错误不会“倒着走”回已经跳过的处理层。
- **所有中间件都调用 `next()`**：鉴权失败或参数错误时应该直接响应，否则非法请求仍会进入业务逻辑。

理解中间件之后，Express 就不再是一堆孤立方法。它更像一条按注册顺序组织的处理链，每一层都可以读取上下文、补充信息、终止请求或把控制权交给下一层。

---

## 八、完整实战：模块化用户 API

前面的示例分别解释了某一个概念。现在把它们组合成一个可以独立运行的小项目。数据暂时保存在内存中，重点是理解模块边界、参数来源、中间件顺序和错误响应，而不是持久化。

### 1. 目录结构

```text
node-learning-api/
├─ app.js
├─ package.json
└─ routes/
   └─ users.js
```

安装与启动：

```bash
npm init -y
npm install express@5.2.1
node app.js
```

### 2. 接口约定

| 方法 | 地址 | 输入 | 成功状态码 | 用途 |
|------|------|------|------------|------|
| GET | `/api/users` | `?keyword=` | 200 | 查询用户列表 |
| GET | `/api/users/:id` | 路径参数 `id` | 200 | 查询单个用户 |
| POST | `/api/users` | JSON 或表单中的 `name` | 201 | 创建用户 |
| GET | `/api/users/debug/sync-error` | 无 | 500 | 验证同步错误处理 |
| GET | `/api/users/debug/async-error` | 无 | 500 | 验证异步错误处理 |

业务响应统一为：

```json
{
  "code": 0,
  "message": "请求成功",
  "data": {}
}
```

这里的 `code` 是应用层字段，不能替代 HTTP 状态码。客户端仍应先根据 200、201、400、404、500 等状态码判断请求结果。

### 3. 用户 Router

```js
// routes/users.js
const express = require('express')

const router = express.Router()

const users = [
  { id: 1, name: 'Ada' },
  { id: 2, name: 'Linus' },
  { id: 3, name: 'Grace' }
]

router.get('/debug/sync-error', (req, res) => {
  throw new Error('用于验证同步错误处理')
})

router.get('/debug/async-error', async (req, res) => {
  await Promise.reject(new Error('用于验证异步错误处理'))
})

router.get('/', (req, res) => {
  const keyword = String(req.query.keyword ?? '').trim().toLowerCase()
  const result = keyword
    ? users.filter((user) => user.name.toLowerCase().includes(keyword))
    : users

  res.status(200).json({
    code: 0,
    message: '查询成功',
    data: result
  })
})

router.get('/:id', (req, res) => {
  const id = Number(req.params.id)

  if (!Number.isInteger(id) || id <= 0) {
    res.status(400).json({
      code: 400,
      message: '用户 ID 必须是正整数',
      data: null
    })
    return
  }

  const user = users.find((item) => item.id === id)

  if (!user) {
    res.status(404).json({
      code: 404,
      message: '用户不存在',
      data: null
    })
    return
  }

  res.status(200).json({
    code: 0,
    message: '查询成功',
    data: user
  })
})

router.post('/', (req, res) => {
  const name = typeof req.body?.name === 'string'
    ? req.body.name.trim()
    : ''

  if (!name) {
    res.status(400).json({
      code: 400,
      message: 'name 是必填字段',
      data: null
    })
    return
  }

  const user = {
    id: users.length === 0 ? 1 : users[users.length - 1].id + 1,
    name
  }

  users.push(user)

  res.status(201).json({
    code: 0,
    message: '创建成功',
    data: user
  })
})

module.exports = router
```

调试路由写在 `/:id` 之前并非随意安排。如果把动态路由放在最前面，`/debug/sync-error` 可能先被当成 `id = 'debug'` 的请求处理。越具体的路径通常应放在越宽泛的动态路径之前。

### 4. 应用入口

```js
// app.js
const express = require('express')
const usersRouter = require('./routes/users')

const app = express()
const port = 3000

app.use(express.json({ limit: '100kb' }))
app.use(express.urlencoded({ extended: false, limit: '100kb' }))

app.use((req, res, next) => {
  const startedAt = process.hrtime.bigint()

  res.on('finish', () => {
    const elapsedNanoseconds = process.hrtime.bigint() - startedAt
    const elapsedMilliseconds = Number(elapsedNanoseconds) / 1e6

    console.log(
      `${req.method} ${req.originalUrl} ${res.statusCode} ${elapsedMilliseconds.toFixed(2)}ms`
    )
  })

  next()
})

app.get('/health', (req, res) => {
  res.status(200).json({
    code: 0,
    message: '服务正常',
    data: null
  })
})

app.use('/api/users', usersRouter)

app.use((req, res) => {
  res.status(404).json({
    code: 404,
    message: '接口不存在',
    data: null
  })
})

app.use((err, req, res, next) => {
  console.error(err)

  if (res.headersSent) {
    next(err)
    return
  }

  const invalidJson = err instanceof SyntaxError && err.status === 400 && 'body' in err

  res.status(invalidJson ? 400 : 500).json({
    code: invalidJson ? 400 : 500,
    message: invalidJson ? 'JSON 格式不合法' : '服务器内部错误',
    data: null
  })
})

app.listen(port, () => {
  console.log(`API running at http://127.0.0.1:${port}`)
})
```

这份入口文件的顺序体现了完整处理链：

```text
请求体解析
  → 请求耗时日志
  → 健康检查或 usersRouter
  → 没有匹配时返回 404
  → 出错时进入错误处理中间件
```

404 处理本质上也是普通中间件：只有前面的路由都没有结束响应，它才会执行。错误处理中间件则只处理传入错误链的请求。

### 5. 使用 curl 验证接口

查询所有用户：

```bash
curl "http://127.0.0.1:3000/api/users"
```

按名称筛选：

```bash
curl "http://127.0.0.1:3000/api/users?keyword=ad"
```

查询单个用户：

```bash
curl "http://127.0.0.1:3000/api/users/1"
```

提交 JSON：

```bash
curl -X POST "http://127.0.0.1:3000/api/users" \
  -H "Content-Type: application/json" \
  -d '{"name":"Brendan"}'
```

在 PowerShell 中也可以使用：

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri 'http://127.0.0.1:3000/api/users' `
  -ContentType 'application/json' `
  -Body '{"name":"Brendan"}'
```

提交 URL-encoded 表单：

```bash
curl -X POST "http://127.0.0.1:3000/api/users" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "name=Ryan"
```

还应主动验证失败路径：

```bash
# 缺少 name，应返回 400
curl -i -X POST "http://127.0.0.1:3000/api/users" \
  -H "Content-Type: application/json" \
  -d '{}'

# 用户不存在，应返回 404
curl -i "http://127.0.0.1:3000/api/users/999"

# 路由不存在，应返回 404
curl -i "http://127.0.0.1:3000/not-found"

# JSON 不合法，应返回 400，而不是 HTML 错误页
curl -i -X POST "http://127.0.0.1:3000/api/users" \
  -H "Content-Type: application/json" \
  -d '{"name":}'

# Express 5 中同步、异步错误都应进入统一错误处理
curl -i "http://127.0.0.1:3000/api/users/debug/sync-error"
curl -i "http://127.0.0.1:3000/api/users/debug/async-error"
```

### 实战复盘

原始接口练习已经具备 Router、查询参数和 JSON 响应的雏形，但“能返回数据”只是第一步。一个更完整的接口还需要做到：

- 方法和路径能够准确表达操作对象。
- 输入来自哪里一目了然，并在进入业务逻辑前校验。
- 创建、参数错误、资源不存在和服务端错误使用不同状态码。
- 正常响应与错误响应保持稳定结构。
- Router 只负责一组相关业务，入口文件负责装配整个应用。
- 无论成功、路由不存在还是发生异常，请求最终都能得到响应。

从单个 `router.get('/get')` 到完整处理链，增加的代码并不只是形式。它们共同解决了客户端能否理解结果、开发者能否定位问题，以及项目扩大后是否还能维护的问题。

---

## 九、把整条请求链串起来

回头看整个学习过程，原生 Node.js 和 Express 并不是前后互相替代、学完就丢掉的两套知识。Express 的许多对象和行为，底层仍建立在 Node.js HTTP 请求与响应之上。

一次 Express 请求可以概括为：

```text
客户端
  ↓ 发送 HTTP 请求
Node.js HTTP 服务器
  ↓
Express 应用
  ↓
全局中间件（解析请求体、日志、鉴权等）
  ↓
Router 路由匹配
  ↓
局部中间件与业务处理函数
  ↓
成功响应 / 404 / 错误处理中间件
  ↓
客户端收到状态码、响应头和响应体
```

其中有几组概念尤其值得区分：

| 容易混淆的概念 | 正确理解 |
|----------------|----------|
| URL 与文件路径 | URL 面向 HTTP 客户端，文件路径面向操作系统；必须解析、转换并校验范围 |
| `exports` 与 `module.exports` | 初始指向同一对象，但模块最终导出的是 `module.exports` |
| 路由与中间件 | 路由按方法和路径处理业务；中间件是更通用的处理层，两者都位于请求链中 |
| `query`、`params`、`body` | 分别来自查询字符串、动态路径和请求体 |
| `next()` 与 `return` | `next()` 把控制权交给后续层；`return` 只结束当前 JavaScript 函数 |
| 语法错误与运行时错误 | `node --check` 只能发现语法问题，变量拼写和请求挂起要通过实际请求暴露 |
| HTTP 状态码与业务 `code` | 状态码属于协议；业务字段属于应用响应，两者不能互相替代 |

---
