# Express学习解析-路由篇

## Express 简介

Express 是一个简洁而灵活的 node.js Web应用框架, 提供了一系列强大特性帮助你创建各种 Web 应用，和丰富的 HTTP 工具。
使用 Express 可以快速地搭建一个完整功能的网站。

[Express官网](http://www.expressjs.com.cn/) 也是一个非常友好的文档。

Express 框架核心特性：

 - 路由: 定义了路由表用于执行不同的 HTTP 请求动作。
 - 中间件: 可以设置中间件来响应 HTTP 请求。
 - 模板引擎: 可以通过向模板传递参数来动态渲染 HTML 页面。

> 这里先从Express路由开始学习解析，再继续它的其他核心特性进行学习与探索。

## 安装 Express (V4.16.2)

```
npm i express -S
```

## 一个简单的 Express 路由

1.创建一个接收get请求的服务器

```
// 1.get.js
const express = require('express');
const app = express();

const port = 8080;

// path 是服务器上的路径，callback是当路由匹配时要执行的函数。
app.get('/', function(req,res){ // 注册一个get请求路由
    res.end('hello express!'); // 结束响应
});

app.listen(port,function(){ // 开启监听端口
    console.log(`server started on port ${port}`);
});

```

2.执行上面代码

[nodemon](http://nodemon.io/)是一种实用工具，将为您的源的任何变化并自动重启服务器监控。
```

$ nodemon 1.get.js

浏览器访问 localhost:8080 结果如下:
hello express!

若访问一个未处理的路由路径 如localhost:8080/user 结果如下:
Cannot GET /user  也就是我们常见的404 Not Found

```

## 根据上面测试案例自己实现一个express

创建一个express-1.0.js
```
// express-1.0.js
// 1.准备一个路由容器来存放 注册的路由
const router = [{ // 这是一个默认路由 当所有注册路由未匹配时 用来处理404请求的响应
        path: '*', // '*' 通配符
        method: '*',
        handler(re  q, res) {
            res.end(`Cannot ${req.method} ${req.url}`)
        }
    }]
```

require的express函数实际上是内部导出的createApplication函数 由它返回一个app对象
```
// express-1.0.js

// 依赖的node模块
const http = require('http'); // http模块 用来搭建web服务器
const url = require('url'); // 解析req.url

function createApplication() { // express内部函数
    return { // 这里先创建一个关于get请求的对象方法

        get(path, handler) { // app.get 用来注册get请求的路由
            router.push({
                path, // 路由匹配路径
                handler, // 路由处理函数
                method: 'get' // get请求方法
            });
        },
        listen() { // app.listen 启动server 监听指定端口

            // express也是需要http搭建web服务器
            const server = http.createServer((req, res) => { // 在该监听函数里进行路由处理
                const { pathname } = url.parse(req.url, true); // 解析出请求路径

                // 根据请求路由和路由容器里的路由进行匹配
                // i = 1 从第二条路由开始匹配 因为第一个路由是处理404的默认路由
                for(let i = 1, l = router.length; i++) {
                    let {path, method, handler} = router[i];

                    // path路径匹配和method请求方法都匹配 则路由匹配成功
                    if (path === pathname && method === req.method.toLowerCase()) {
                        return handler(req, res); // 执行对应路由处理函数 并停下继续往下匹配
                    }
                }
                // 若上面路由都没有匹配成功 则执行默认路由来进行处理
                router[0].handler(req, res);
            });

            // 开启服务器进行监听 app.listen相当于代理了server.listen
            server.listen.apply(server, arguments);
        }
    }
}

module.exports = createApplication; // 导出这个express模块
```

在上面第一个demo中换成导入express-1.0.js 查看效果

```
const express = require('./lib/express-1.0.js');
```

## 路由句柄

可以为请求处理提供多个回调函数，每个函数里都有一个next方法 调用next方法可将控制权交给剩下的路径，其行为类似 中间件。
路由句柄有多种形式，可以是一个函数、一个函数数组 ，或者是两者混合 也可以链式 如下：

使用多个回调函数处理/get路由（需要指定next 才会进入第二个的回调中）

```
app.get('/get', function (req, res, next) {
  console.log('Hello one!');
  next(); // 调用next 才会进入下面的回调中
}, function (req, res) {
  res.send('Hello two!');
});
```

上面多函数也可以写成数组形式：

```
let cb1 =  function (req, res, next) {
  console.log('Hello one!');
  next();
};

let cb2 = function (req, res, next) {
  res.send('Hello two!');
}

app.get('/get', [cb1, cb2]);
```

```
// Restfule风格设计
app.get('/get', function (req, res, next) {
  console.log('get1');
  next();
}, function (req, res) {
  res.end('get2');
}).post('/post', (req, res) => {
  res.end('post');
}).put('/put', (req, res) => {
  res.end('put');
}).delete('/delete', (req, res) => {
  res.end('delete');
});
```

在路由运行过程中，Router是一个路由系统容器，根据请求方法路由规则分为几层 里面存放Route 称为layer, Route里也存放着一层层的路由处理方法回调，Router和Route都各自维护了一个stack数组，该数组就是用来存放中间件和路由的。
![](assets/markdown-img-paste-2018031920011734.png)

## 改造下我们之前写的application.js

```
const http = require('http');
const url = require('url');
const Router = require('./router'); // 这里我们需要一个Router容器

function Application() { // 抽象为Application类 将get listen方法挂载到原型上
    this._router = new Router(); // 实例化一个router
}

Application.prototype.get = function(path, handler) {
    this._router.push({
        path,
        handler,
        method: 'get'
    })
}

Application.prototype.listen = function() {
    let that = this;
    const server = http.createServer((req, res) => {
        const {pathname} = url.parse(req.url, true); // 请求路径
        for (let i = 1, len = that._router.length; i < len; i++) {
            let {path, method, handler} = that._router[i];
            if (path === pathname && method === req.method.toLowerCase()) { // 匹配路由路径
                return handler(req, res); // 路由匹配上就不会继续匹配下去
            }
        }
    });
    server.listen.apply(server, arguments);
}

module.exports = Application;
```

## 创建application中需要的Router对象

```
// router.js

function Router() {
    function router(req, res, next) {
        router.handle(req, res, next);
    }
    Object.setPrototypeOf(router, proto);
    router.stack = [];
    //声明一个对象，用来缓存路径参数名它对应的回调函数数组
    router.paramCallbacks = {};
    //在router一加载就会加载内置 中间件
    // query
    router.use(init);
    return router;
}

```
