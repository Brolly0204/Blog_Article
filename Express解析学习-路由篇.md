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

## Express简单使用

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

## 根据上面测试案例自己实现一个简单express

### express简单实现

```
// express-1.0.js
const http = require('http');
const url = require('url');

function createApplication() {

    function app(req, res) {
        app.handle(req, res);
    }

    app.routes = []; // 路由容器
    app.get = function(path, handle) { // 注册get方法的路由
        app.routes.push({ // 路由表
            path,
            handle,
            method: 'get'
        });
    }

    app.listen = function() { // 创建http服务器
        const server = http.createServer(app);
        server.listen.apply(server, arguments);
    }

    app.handle = function(req, res) { // 路由处理
        let { pathname } = url.parse(req.url); // 获取请求路径
        let ids = 0;
        let routes = this.routes; // 获取路由表
        function next() { // next控制路由进入下一匹配
            if (ids >= routes.length) {
                return res.end(`Cannot ${req.method} ${pathname}`);
            }

            let {path, method, handle} = routes[ids++];
            // 进行路由匹配
            if ((pathname === path || pathname === '*') && method === req.method.toLowerCase()) {
                return handle(req, res)
            } else { // 如果不匹配 则去下一个路由匹配
                next();
            }
        }
        next();
    }

    return app;
}

module.exports = createApplication;
```

### 导入我们自己的express.js
```

const express = require('./express-1.0.js');
const app = express();

app.get('/get', function(req, res) {
    res.send('hello express');
});
```
下面我们开始进行源码分析

## 源码分析

express源码目录结构

![](https://user-gold-cdn.xitu.io/2018/3/24/16254195ec260680?w=328&h=502&f=png&s=25802)

## 路由系统
对于路由中间件，在整个个Router路由系统中stack 存放着一个个layer, 通过layer.route 指向route路由对象, route的stack的里存放的也是一个个layer，每个layer中包含(method/handler)。

![](https://user-gold-cdn.xitu.io/2018/3/24/162541b0548860b4?w=1436&h=738&f=png&s=113014)

在源码里面主要涉及到几个类和方法

```
createApplicaton
Application(proto)
Router
Route
Layer
```

1. express()返回一个app
实际上express指向内部createApplication函数 函数执行返回app

```
var mixin = require('merge-descriptors'); // merge-descriptors是第三方模块，合并对象的描述符
var proto = require('./application'); // application.js中 导出 proto

function createApplicaton() { // express()
	var app = function(req, res, next) { // app是一个函数
        app.handle(req, res, next); // 处理路由
    };

    mixin(app, EventEmitter.prototype, false); // 将EventEmitter.prototype的对象属性合并到app上一份
    mixin(app, proto, false); // 将proto中的挂载的一些属性方法 如 app.get app.post 合并到app上
    app.init(); // 初始化 声明一些私有属性
    return app;
}
```

app上的很多属性方法都来自application.js导出的proto对象， 在router/index.js中 也有一个命名为proto的函数对象 挂载了一些静态属性方法 proto.handle proto.param proto.use ...

```
// router/index.js
var proto = module.exports = function(options) {}

```


在application.js 导出的proto中 挂载这合并到app上的请求方法，源码如下：

```
// application.js

// methods是一个数组集合[get, post, put ...]，里面存放了一系列http请求方法 通过遍历给app上挂载一系列请求方法，即app.get()、app.post() 等
methods.forEach(function(method){
  app[method] = function(path){
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter(); // 创建一个Router实例 this._router = new Router();

    var route = this._router.route(path); // 对应router/index 中的 proto.route
    route[method].apply(route, slice.call(arguments, 1)); // 添加路由中间件 下面详细讲解
    return this;
  };
});
```

this.lazyrouter 用来创建一个Router实例 并挂载到 app._router

```
// application.js
methods.forEach(function(method){
  app[method] = function(path){

    this.lazyrouter(); // 创建一个Router实例 this._router = new Router();
  }
});

app.lazyrouter = function lazyrouter() {
  if (!this._router) {
    this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });

    this._router.use(query(this.get('query parser fn')));
    this._router.use(middleware.init(this));
  }
};
```

## 注册中间件

可以通过两种方式添加中间件：app.use用来添加非路由中间件，app[method]添加路由中间件，这两种添加方式都在内部调用了Router的相关方法来实现：

### 注册非路由中间件
```
// application.js

app.use = function(fn) { // fn 中间件函数 有可能是[fn]
    var offset = 0;
    var path = '/';

    var fns = flatten(slice.call(arguments, offset)); // 展平数组
    // setup router
    this.lazyrouter(); // 创建路由对象
    var router = this._router;

    fns.forEach(function(fn) {
      router.use(path, function mounted_app(req, res, next) { // app.use 底层调用了router.use
      var orig = req.app;
      fn.handle(req, res, function (err) {
        setPrototypeOf(req, orig.request)
        setPrototypeOf(res, orig.response)
        next(err);
      });
    });
}

```
通过上面知道了 app.use底层调用了router.use 接下来我们再来看看router.use

```
// router/index.js
var Layer = require('./layer');

proto.use = function(fn) {
    var offset = 0; // 参数偏移值
    var path = '/'; // 默认路径 /  因为 use

    // 第一个参数有可能是path 所以要从第二个参数开始截取
    if (typeof arg !== 'function') {
         offset = 1;
         path = fn;
    }

    // 展平中间件函数集合 如中间件函数是以数组形式注册的如[fn, fn] 当slice截取时会变成[[fn, fn]] 所以需要展平为一维数组
    var callbacks = flatten(slice.call(arguments, offset)); // 截取中间件函数

    for(var i = 0; i < callbacks.length; i++) { // 遍历每一个中间件函数
        var fn = callbacks[i];
        if (typeof fn !== 'function') { // 错误提示
          throw new TypeError('Router.use() requires a middleware function but got a ' + gettype(fn))
        }

        // 添加中间件
        // 实例化一个layer 对象
        var layer = new Layer(path, {
            sensitive: this.caseSensitive,
            strict: false,
            end: false
        }, fn);

        // 非路由中间件，该字段赋值为undefined
        layer.route = undefined;
        this.stack.push(layer); // 将layer添加到 router.stack
    }
}
```

### 注册路由中间件

在上面application.js 中的app对象 添加了很多http关于请求  app[method]就是用来注册路由中间件

```
// application.js

var app = exports = module.exports = {};

var Router = require('./router');
var methods = require('methods');

methods.forEach(function(method){
  app[method] = function(path){ // app上添加 app.get app.post app.put 等请求方法

    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter(); // 创建Router实例对象 并赋给this._router

    // 调用this._router.route => router/index.js中的proto.route
    var route = this._router.route(path);
    route[method].apply(route, slice.call(arguments, 1)); // 调用router中对应的method方法 注册路由
    return this;
  };
});

```
在上面 app[method]中底层实际调用了router[method] 也就是 this._router.route[method]

我们看看this._router.route(path); 这段代码发生了什么
```
var route = this._router.route(path); // 里面创建了一个route实例 并返回
route[method].apply(route, slice.call(arguments, 1)); // 调用route中的对象的route[method]
```

router 中的 this._router.route  创建一个route对象 生成一个layer 将path 和 route.dispatch 传入layer, layer的route指向 route对象 将Router和Route关联来起来， 最后把route对象作为返回值
```
// router/index.js

var Route = require('./route');
var Layer = require('./layer');

proto.route = function (path) {
    var route = new Route(path); // app[method]注册一个路由 就会创建一个route对象

    var layer = new Layer(path, { // 生成一个route layer
      sensitive: this.caseSensitive,
      strict: this.strict,
      end: true
    }, route.dispatch.bind(route)); // 里将生成的route对象的dispatch作为参数传给layer里面

    // 指向刚实例化的路由对象（非常重要），通过该字段将Router和Route关联来起来
    layer.route = route;

    this.stack.push(layer); // 将layer添加到Router的stack中
    return route; // 将生成的route对象返回
}

```
对于路由中间件，路由容器中的stack（Router.stack）里面的layer通过route字段指向了路由对象，那么这样一来，Router.stack就和Route.stack发生了关联，关联后的示意模型如下图所示：


![](https://user-gold-cdn.xitu.io/2018/3/24/162541b9e6069980?w=786&h=186&f=png&s=28655)


app[method]中 调用 route[method].apply(route, slice.call(arguments, 1));

```
// router/index.js

// 实际上 application.js 中 app[method] 调用的是 router对象中对应的http请求方法 额外添加了一个all方法
methods.concat('all').forEach(function(method){
  proto[method] = function(path){
    var route = this.route(path) // 返回创建的route对象
    route[method].apply(route, slice.call(arguments, 1)); // router[method] 又调用了 route[method]
    return this;
  };
});

```

最后我们来看下 router/route.js 中的Route

```
// router/route.js

function Route(path) { // Route类
  this.path = path;
  this.stack = []; // route的stack

  debug('new %o', path)
  this.methods = {}; // 用于各种HTTP方法的路由处理程序
}

var Layer = require('./layer');
var methods = require('methods');

// 又是相同的一段代码 在给Route的原型上添加http方法
methods.forEach(function(method){
  Route.prototype[method] = function(){
    var handles = flatten(slice.call(arguments)); // 传递进来的处理函数

    for (var i = 0; i < handles.length; i++) {
      var handle = handles[i];

      if (typeof handle !== 'function') {
        var type = toString.call(handle);
        var msg = 'Route.' + method + '() requires a callback function but got a ' + type
        throw new Error(msg);
      }

      debug('%s %o', method, this.path)

      var layer = Layer('/', {}, handle); // 在route中也有layer 里面保存着 method和handle
      layer.method = method;

      this.methods[method] = true; // 标识 存在这个method的路由处理函数
      this.stack.push(layer); // 将layer 添加到route的stack中
    }

    return this; // 将route对象返回
  };
});
```


Route 中的all方法

```
Route.prototype.all = function all() {
  var handles = flatten(slice.call(arguments));

  for (var i = 0; i < handles.length; i++) {
    var handle = handles[i];

    if (typeof handle !== 'function') {
      var type = toString.call(handle);
      var msg = 'Route.all() requires a callback function but got a ' + type
      throw new TypeError(msg);
    }

    var layer = Layer('/', {}, handle);
    layer.method = undefined; // all 匹配所以方法

    this.methods._all = true; // all方法标识
    this.stack.push(layer); // 添加到route的stack中
  }

  return this;
};
```

最终路由注册关系链 app[method] => router[method] => route[method] 最终在route[method]里完成路由注册


### 接下来我们看看Layer

```
// route/layer.js
var pathRegexp = require('path-to-regexp');

module.exports = Layer;

function Layer(path, options, fn) {
    if (!(this instanceof Layer)) {
        return new Layer(path, options, fn);
    }

    this.handle = fn; // 存储处理函数
    this.regexp = pathRegexp(path, this.keys = [], opts); // 根据路由路径 生成路由规则正则 用来路由匹配
}

Layer.prototype.match = function(path) { // 请求路径 是否匹配 该层路由路径规则this.regexp

}
```


## 启动server

```
// application.js
var http = require('http');

app.listen = function listen() {
  var server = http.createServer(this); // this => express.js 中的 app
  return server.listen.apply(server, arguments);
};

// express.js 中的 app
var app = function(req, res, next) {
  app.handle(req, res, next);
};

```

## 路由调用

app.handle 调用this._router.handle 进行路由处理

```
express.js
function createApplicaton() {
    let app = function(req, res, next) { // 持续监听请求
        app.handle(req, res, next); // 路由处理函数
    }
}

```

express.js中的app.handle 实际来自application.js的app.handle 底层调用router.handle
```
// application.js

app.handle = function handle(req, res, callback) {
  var router = this._router;

  // final handler
  var done = callback || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  router.handle(req, res, done); // 底层调用router.handle
};
```

router.handle 调用 内部next函数 在router的stack中寻找匹配的layer

```
// router/index.js


function matchLayer(layer, path) {
  try {
    return layer.match(path);
  } catch (err) {
    return err;
  }
}


proto.handle = function(req, res, done) {
    // middleware and routes
    var stack = self.stack; // router的stack 里面存放着中间件和路由
    var idx = 0;

    next();
    function next(err) {

        if (idx >= stack.length) { // 边界判断
             setImmediate(done, layerError);
             return;
        }

        // 获取请求路径
        var path = getPathname(req);

        var layer;
        var match;
        var route;
        while (match !== true && idx < stack.length) { // 一层层进行路由匹配
          layer = stack[idx++]; // 从router的stack取出没一个layer

          match = matchLayer(layer, path); // matchLayer调用 该layer的match方法进行路由路径进行匹配

          route = layer.route; // 得到关联的route对象
          var method = req.method; // 获取请求方法

          var has_method = route._handles_method(method); // 调用route的_handles_method返回Boolean值

          if (match !== true) { // 如果没匹配上处理
            return done(layerError);
          }

          if (route) { // 调用layer的handle_request 处理请求 执行handle
             return layer.handle_request(req, res, next);
          }
        }
    }
}
```

路由处理 app.handle => router.handle => layer.handle_request

layer的handle_request 调用next一次获取route stack中的处理方法

```

Layer.prototype.handle_request = function handle(req, res, next) {
  var fn = this.handle;  // 获取new Layer时保存的handle
    // function Layer() {
        // this.handle = fn;
    // }

  if (fn.length > 3) {
    // not a standard request handler
    return next();
  }

  try {
    fn(req, res, next);
  } catch (err) {
    next(err);
  }
};
```

## 小结
app.use用来添加非路由中间件，app[method]添加路由中间件，中间件的添加需要借助Router和Route来完成，app相当于是facade，对添加细节进行了包装。

Router可以看做是一个存放了中间件的容器。对于里面存放的路由中间件，Router.stack中的layer有个route属性指向了对应的路由对象，从而将Router.stack与Route.stack关联起来，可以通过Router遍历到路由对象的各个处理程序。
