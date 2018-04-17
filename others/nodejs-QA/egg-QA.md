#### 1.egg-mysql需要指定主键来更新
报错信息如下:
<pre>
unhandledRejectionError: Can not auto detect update condition, please set options.where, or make sure obj.id exists
</pre>
解决方法:
```js
 yield scope.app.mysql.update("data_json_markdown", {
    id:componentMap.id,
    //这里必须指定，对应于数据的primary key
    gmt_create: new Date(),
    gmt_modified: new Date()
    })
```

#### 2.egg不管如何对于查询(Query)操作都是返回空但是不报错
比如下面的例子:
```js
function processData(scope,x,x){
  var componentMap = yield scope.app.mysql.get('data_json_markdown', { id: 42 });
  //此时你无法再访问数据库了
}
var update = function(scope) {
  //自动执行我们的regenerator函数
co.wrap(function*(scope) {
   //代码片段1
   //var componentMap = yield scope.app.mysql.get('data_json_markdown', { id: 42 });
   //在这里直接yield是可以访问到数据库的
   //下面是代码片段2
   //processData(scope,x,x)
 })
}
module.exports = function*() {
  update(this);
};
```
总之，在egg中如果要访问到数据库，必须通过`yield`来完成，而通过外部的processData函数直接拿到scope这种方式是不行的，后续再对这一部分内容进行深入研究。包括重新开启一个Worker进程，传入scope这种方式操作数据库都是不支持的!!!

#### 3.egg无法使用aync和await
原因是: egg-mysql包装了`ali-rds`后者不支持promise所以不能使用async/await。但是给出了解决方法,即用co包装:
```js
// app.js
const co = require('co');
module.exports = app => {
  app.beforeStart(async () => {
      ['insert', 'get', 'select', 'update', 'delete', 'query'].forEach(func => {
        app.mysql[func] = co.wrap(app.mysql[func]);
      });
    });
};
```
这样的话，我们的[`co.wrap`](https://github.com/tj/co#var-fn--cowrapfn)方法就会将app.mysql上的insert,get,select等方法包装成为支持promise的形式。

#### 4.如何在nodejs中使用ES6语法
解答:目前nodejs已经原生支持了很多ES6语法了，具体支持的程度你可以[点击这里查看](http://taobaofed.org/blog/2016/01/07/find-back-the-lost-es6-features-in-nodejs/)，当然其并不是支持所有的ES6新特性，所以你可以通过下面方式来完成:

首先:在项目根目录添加.babelrc文件:
```js
{
  "plugins": [
    "transform-strict-mode",
    "transform-es2015-modules-commonjs",
    "transform-es2015-spread",
    "transform-es2015-destructuring",
    "transform-es2015-parameters"
  ]
}
```
然后:使用babel-node替代babel进行打包:
```js
babel-node src/index.js
```
其中[babel-node](http://babeljs.io/docs/usage/cli/)会在执行代码之前对ES6代码进行编译，这样的话，我们在nodejs端就可以使用ES6的代码规范了。

#### 5.koa的洋葱圈模型
假如下面的代码:
```js
const one = (ctx, next) => {
  console.log('>> one');
  next();
  console.log('<< one');
}
const two = (ctx, next) => {
  console.log('>> two');
  next();
  console.log('<< two');
}
const three = (ctx, next) => {
  console.log('>> three');
  next();
  console.log('<< three');
}

app.use(one);
app.use(two);
app.use(three);
```
那么输出结果为:
<pre>
one
two
three
three
two
one
</pre>
koa是从第一个中间件开始执行，遇到next进入下一个中间件，一直执行到最后一个中间件，在逆序，执行上一个中间件next之后的代码，一直到第一个中间件执行结束才发出响应。如下图:

![](https://segmentfault.com/img/bVUsEI?w=687&h=460)

next就是执行下一个中间件，没有下一个的话就会执行剩下的代码，然后再一层一层出去。也就是说最里层的中间件的next实际上没什么作用。它其实创建了一个`空的promise`而已，这样调用者不用关心链条不小心终端的问题

#### 6.koa的context对象
Koa Context*将node的request和response对象*封装在一个单独的对象里面，其为编写 web 应用和 API 提供了很多有用的方法。这些操作在 HTTP 服务器开发中经常使用，因此其被添加在上下文这一层，而不是更高层框架中，因此将迫使中间件需要重新实现这些常用方法。context在每个 request 请求中被创建，在中间件中作为接收器(receiver)来引用，或者通过 this 标识符来引用：
```js
app.use(function *(){
  this; // is the Context
  this.request; // is a koa Request
  this.response; // is a koa Response
});
```
许多context的访问器和方法为了便于访问和调用，简单的委托给他们的ctx.request和ctx.response(Koa的request和response对象)所对应的等价方法， 比如说ctx.type和ctx.length就是指的是response对象中对应的方法，ctx.path和ctx.method指的就是request对象中对应的方法。这就是官网的文档中明明展示的是request.is但是实际上调用的是cxt.is,因为两者是等价的：
```js
// With Content-Type: text/html; charset=utf-8
ctx.is('html'); // => 'html'
ctx.is('text/html'); // => 'text/html'
ctx.is('text/*', 'text/html'); // => 'text/html'
// When Content-Type is application/json
ctx.is('json', 'urlencoded'); // => 'json'
ctx.is('application/json'); // => 'application/json'
ctx.is('html', 'application/*'); // => 'application/json'
ctx.is('html'); // => false
```
如果要获取node中的request对象可以通过*ctx.req*，而获取node中的response可以获取*ctx.res*。但是他们和Koa本身的request对象和reponse对象是不一样的，后者是通过"ctx.request"和"ctx.response"来获取!


#### 7.egg中如何配置数据库连接池信息
你必须知道[egg-mysql](https://github.com/eggjs/egg-mysql)下直接封装了[ali-rds](https://github.com/ali-sdk/ali-rds),而后者直接封装了[mysql](https://github.com/mysqljs/mysql#pool-options)。假如你的egg的mysql配置内容如下:
```js
exports.mysql = {
    // 数据库信息配置
    client: {
      host: 'tddl.daily2.alibaba.net',
      port: 3306,
      user: 'POLY_APP',
      // connectionLimit : 10,
      //其中connectionLimit是_ali-rds@3.0.1@ali-rds默认的值，默认为5
      // acquireTimeout:20000,
      password: '123456',
      database: 'POLY_APP',
      // 是否启用加密密码
      encryptPassword: false,
    },
    // 是否加载到 app 上，默认开启
    app: true,
    // 是否加载到 agent 上，默认关闭
    agent: false,
  };
```
很可能会抛出如下的超时错误:
<pre>
  2017-11-15 08:58:25,029 ERROR 189112 nodejs.PROTOCOL_SEQUENCE_TIMEOUTError: [egg-schedule] egg-schedule:/home/admin/polymerization/target/polymerization/app/sched
ule/grob2DB.js excute error. Handshake inactivity timeout
    at Handshake.<anonymous> (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/protocol/Protocol.js:164:17)
    at emitNone (events.js:67:13)
    at Handshake.emit (events.js:166:7)
    at Handshake._onTimeout (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/protocol/sequences/Sequence.js:129:8)
    at Timer.listOnTimeout (timers.js:96:15)
    --------------------
    at Protocol._enqueue (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/protocol/Protocol.js:145:48)
    at Protocol.handshake (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/protocol/Protocol.js:52:23)
    at PoolConnection.connect (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/Connection.js:130:18)
    at Pool.getConnection (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/Pool.js:48:16)
    at /home/admin/polymerization/target/polymerization/node_modules/_pify@2.3.0@pify/index.js:29:7
    at Pool.<anonymous> (/home/admin/polymerization/target/polymerization/node_modules/_pify@2.3.0@pify/index.js:12:10)
    at Pool.ret [as getConnection] (/home/admin/polymerization/target/polymerization/node_modules/_pify@2.3.0@pify/index.js:56:34)
    at Pool.releaseConnection (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/Pool.js:157:10)
    at Pool._removeConnection (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/Pool.js:277:8)
    at Pool._purgeConnection (/home/admin/polymerization/target/polymerization/node_modules/_mysql@2.15.0@mysql/lib/Pool.js:258:8)
</pre>

此时添加上面的connectionLimit和acquireTimeout参数就可以了(其实只要添加后者就行)。此时如果在[mysql](https://github.com/mysqljs/mysql#pool-options)的Pool.js中打印创建连接池的配置参数将会得到如下的内容:
```js
{
    "config": {
        "acquireTimeout": 20000,
        //https://github.com/mysqljs/mysql#pool-options
        //这里是从连接池获取一个连接超时时间，和connectTimeout有区别，因为获取一个连接不代表会真实使用它
        "connectionConfig": {
            "host": "tddl.daily2.alibaba.net",
            "port": 3306,
            "user": "POLY_APP",
            "password": "123456",
            "database": "POLY_APP",
            "connectTimeout": 20000,
            //获取一个可以使用的连接的超时设置
            "insecureAuth": false,
            "supportBigNumbers": false,
            "bigNumberStrings": false,
            "dateStrings": false,
            "trace": true,
            "stringifyObjects": false,
            "timezone": "local",
            "flags": "",
            "ssl": false,
            "multipleStatements": false,
            "typeCast": true,
            "maxPacketSize": 0,
            "charsetNumber": 33,
            "clientFlags": 455631
        },
        "waitForConnections": true,
        "connectionLimit": 10,
        // 其中connectionLimit是_ali-rds@3.0.1@ali-rds默认的值，默认为5
        "queueLimit": 0
    }
}
```

参考资料:

[egg-mysql无法访问数据库](https://github.com/eggjs/egg/issues/647)

[co.wrap将generator函数包装成为普通函数](https://github.com/tj/co)

[egg官方文档](https://eggjs.org/zh-cn/tutorials/async-function.html#%E8%B0%83%E7%94%A8-generator-function-api)

[找回 Node.js 里面那些遗失的 ES6 特性](http://taobaofed.org/blog/2016/01/07/find-back-the-lost-es6-features-in-nodejs/)

[egg官方文档](https://eggjs.org/zh-cn/basics/router.html)

[koa官方文档](http://koajs.com/)
