> 为了更好的分工合作，让前端能在不依赖后端环境的情况下进行开发，其中一种手段就是为前端开发者提供一个web容器，这个本地环境就是 mock-server。

数据mock可以有两种思路：

* 在 client 端mock
* 在 server 端mock

第一种方式拦截了请求的发出，直接返回 mock 的数据，而第二种方式请求则真实地发出，只是在 server 端进行 route 拦截。然而身为一名有“尊严”的前端怎么能去求后端呢？所以我们毫不犹豫的选择第一种方式。

目前很多前端 mock 数据的方案的基本流程都是使用 node.js 来模拟 http 请求，配置 router 返回 mock 数据。

让我们设想一下一个比较好的 mock-server 该有的能力：

* 与线上环境一致的接口地址，每次构建前端代码时不需要修改调用接口的代码
* **所改即所得**，具有热更新的能力，每次增加/修改 mock 接口时不需要重启 mock 服务，更不用重启前端构建服务
* 能配合 Webpack
* mock 数据可以由工具生成不需要自己手动写
* 能模拟 **POST、GET** 请求
* 简单（包括：文件结构简单、编写代码简单）

所以接下来给大家介绍一下我自己总结下来一套使用起来比较舒服的 mock-server 解决方案，其中也用到了许多工具和框架，在整个搭建过程中自己同时也学习了很多。

大致的主要思路：以 [json-server](https://github.com/typicode/json-server) 作为 mock 服务器， [mock.js](http://mockjs.com/) 生成 mock 数据，利用 gulp + nodemon + browser-sync 监听 mock 文件的改动重启 node 服务，刷新浏览器，以此达到一种相对完美的 mock-server 要求。

#### json-server搭配mock.js

这里以 Webpack 的前端工程为例：

```bash
npm install json-server mockjs --save
```

在项目根目录新建 mock 文件夹，新建 `mock/db.js` 作为 mock 数据源，`mock/server.js` 作为 mock 服务，`mock/routes.js` 重写路由表。

```javascript
// db.js
var Mock = require('mockjs');

module.exports = {
  getComment: Mock.mock({
    "error": 0,
    "message": "success",
    "result|40": [{
      "author": "@name",
      "comment": "@cparagraph",
      "date": "@datetime"
    }]
  }),
  addComment: Mock.mock({
    "error": 0,
    "message": "success",
    "result": []
  })
};
```

这里我们利用 mock.js 生成 mock 数据，可以尽可能的还原真实数据，还可以减少数据构造的复杂度。

```javascript
// routes.js
module.exports = {
  "/comment/get.action": "/getComment",
  "/comment/add.action": "/addComment"
}
```

我们可以通过路由表的配置实现复杂的路由配置，[详细配置规则](https://github.com/typicode/json-server#add-custom-routes)


```javascript
// server.js
const jsonServer = require('json-server')
const db = require('./db.js')
const routes = require('./routes.js')
const port = 3000;

const server = jsonServer.create()
const router = jsonServer.router(db)
const middlewares = jsonServer.defaults()
const rewriter = jsonServer.rewriter(routes)

server.use(middlewares)
// 将 POST 请求转为 GET
server.use((request, res, next) => {
  request.method = 'GET';
  next();
})

server.use(rewriter) // 注意：rewriter 的设置一定要在 router 设置之前
server.use(router)

server.listen(port, () => {
  console.log('open mock server at localhost:' + port)
})
```

现在打开 terminal 输入命令

```bash
$ node mock/server.js
```

打开 http://localhost:3000/comment/get.action 即可查看到我们想要的数据：

![](http://ww3.sinaimg.cn/large/006tNc79gy1fg74g54gx5j31kw0f046l.jpg)

是不是这样就算搭建完了我们的 mock-server ？不，并没有。我们可以尝试修改一下 db.js 的文件内容，刷新浏览器发现 mock 数据并没有像我们想象的那样修改。那也就是说每次当我们需要添加/修改 mock 数据使都需要重启一次 mock 服务。What ？？？

除此之外我们还需要进行端口代理，以至于不与 Webpack 的构建端口产生跨域。

#### 端口代理

通过 Webpack 配置 proxy 代理：

```javascript
module.exports = {
  ...
  devServer: {  
    //其实很简单的，只要配置这个参数就可以了  
    proxy: {  
      '/api/': {  
        target: 'http://localhost:3000',
  	    changeOrigin: true,
        pathRewrite: {
          '^/api': ''
        }
      }
    }
  } 
}
```

接着在代码里进行 ajax 请求就可以写成，这里以 axios 为例子：

```javascript
function getComments () {
  axios.get('api/comment/get.action', {}).then((res) => {
    console.log(res.data)
  })
}
```

#### 文件改动自动刷新

我们希望更改 mock 文件能和 webpack 热更新一样，所改即所得。这里我使用了 nodemon，利用 gulp 建立自动执行的任务。

```bash
npm install gulp gulp-nodemon browser-sync --save
```

gulpfile.js 的代码如下：

```javascript
const path = require('path');
const gulp = require('gulp');
const nodemon = require('gulp-nodemon');
const browserSync = require('browser-sync').create();
const server = path.resolve(__dirname, 'mock');

// browser-sync配置，配置里启动nodemon任务
gulp.task('browser-sync', ['nodemon'], function() {
  browserSync.init(null, {
    proxy: "http://localhost:8080", // 这里的端口和webpack的端口一致
    port: 8081
  });
});

// browser-sync 监听文件
gulp.task('mock', ['browser-sync'], function() {
  gulp.watch(['./mock/db.js', './mock/**'], ['bs-delay']);
});

// 延时刷新
gulp.task('bs-delay', function() {
  setTimeout(function() {
    browserSync.reload();
  }, 1000);
});

// 服务器重启
gulp.task('nodemon', function(cb) {
  // 设个变量来防止重复重启
  var started = false;
  var stream = nodemon({
    script: './mock/server.js',
    // 监听文件的后缀
    ext: "js",
    env: {
      'NODE_ENV': 'development'
    },
    // 监听的路径
    watch: [
      server
    ]
  });
  stream.on('start', function() {
    if (!started) {
      cb();
      started = true;
    }
  }).on('crash', function() {
    console.error('application has crashed!\n')
    stream.emit('restart', 10)
  })
});
```

这样以后我们在构建我们 Webpack 工程时只需要先执行

```bash
$ npm run dev
```

之后新建 terminal 执行

```bash
$ gulp mock
```

就可以搭建一个随改随变的 mock-server 环境，再也不用看后台的脸色啦~

如果有任何问题欢迎留言

个人的 [vue-starter-kit](https://github.com/yanm1ng/vue-starter-kit) 项目也采用了这种 mock 方案，如果有需要也可以参考，欢迎 star 
