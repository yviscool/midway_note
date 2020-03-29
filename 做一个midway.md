# 做一个 midway


## 序言

假设你熟悉 egg , egg 的 load 机制. 以及 egg-router 原理.  那么通过前面几节我们 injection 的剖析, 对照 midway.  我们要实现一个 midway, 怎么办?



## 前置知识

egg load 的机制

egg-router / koa-router 原理

injection 注入依赖 原理



egg 的 load 各个阶段 (建议大致看过源码的阅读以下内容)

```js
'use strict';

const EggLoader = require('egg-core').EggLoader;

/**
 * App worker process Loader, will load plugins
 * @see https://github.com/eggjs/egg-loader
 */
class AppWorkerLoader extends EggLoader {

  /**
   * loadPlugin first, then loadConfig
   * @since 1.0.0
   */
  loadConfig() {
    this.loadPlugin();
    super.loadConfig();
  }

  /**
   * Load all directories in convention
   * @since 1.0.0
   */
  load() {
    // app > plugin > core
    this.loadApplicationExtend();
    this.loadRequestExtend();
    this.loadResponseExtend();
    this.loadContextExtend();
    this.loadHelperExtend();

    this.loadCustomLoader();

    // app > plugin
    this.loadCustomApp();
    // app > plugin
    this.loadService();
    // app > plugin > core
    this.loadMiddleware();
    // app
    this.loadController();
    // app
    this.loadRouter(); // Dependent on controllers
  }

}

module.exports = AppWorkerLoader;
```



midway = egg + ioc ,  ioc 我们已经有了, 我们再看看 egg 有什么. 

有 config , logger , plugin, service,  router.  而 egg 是通过 EggLoader 去框架, 应用, 插件目录下面去 loader 这些信息, 绝大部分最终都会挂载到 ctx/ctx.app = app上面. 反正我们写 egg 的时候, 有个 ctx 基本就无敌天上地下了.

如果你想要在 egg 之上封装一个上层框架, 就免不了跟这个 EggLoader 打交道. 而且你如果想要复用 egg 的生态而且要兼容, 这些 loadXXXX 绝大部分都是动不了的, 很多都是深度绑定的(第三方插件都是依赖这些约束去扩展的). 那么我们能做的只有扩展了.  

插个话题, 笔者之前看 nest 之后, egg 发布了 2.0. 我就想要把 nest 的一些特性移植到 egg 身上. 在看到 EggLoader 的时候, 大概就有了一些想法. 最终形成了一个 egg-pig 插件. 

```js
// app.js
module.exports = () => {

  const defaultLoadController = EggLoader.prototype.loadController;

  EggLoader.prototype.loadController = function() {

    defaultLoadController.call(this);

    if (this.config.eggpig.pig) {

      // extends
      this.createMethodsProxy();
      this.resolveRouters();
    }

  };

  Object.assign(EggLoader.prototype, require('./lib/loader/egg_loader'));

};
```

可以看到主要是利用 loadCustomApp 阶段加载 app.js 的时候, 修改了原型链上的 loadController ,  然后扩展了原型链的一些东西.  这样以比较低的成本新增了许多特性同时兼容了 egg. 

回到主题上,首先 midway 是 egg 的扩展, 利用 ioc 的能力, 我们要实现几块解耦.

* 通过 @controller 和 @get @post 与 router 文件解耦

- 通过 @config 能力，和 app.config 解耦
- 通过 @plugin，和 app.xxx 插件解耦
- 通过 @inject() ctx 和请求链路解耦
-  .....

我们要实现这些功能, 势必就要继承这个 EggLoader 然后自定义 load 方法, 扩展一些方法. 



我们先看看我们最熟悉的路由是怎么解耦的. 我们通过 @Controller @Get @Post 这些来和 router.ts 解耦

```js
// router.ts
export default (app: Application) => {
  const { controller, router } = app;

  // router.get('/', controller.home.index);
};
```

在 loadRouter 阶段, 我们会通过 router.verb 来注册这些路由.  用过 koa-router 的都知道最终有 app.use(router.routes()) . 这个代码在哪里呢.

```js
// egg-core/egg.js
// 上面的 router 
 get router() {
    if (this[ROUTER]) {
      return this[ROUTER];
    }
    const router = this[ROUTER] = new Router({ sensitive: true }, this);
    // register router middleware 
    this.beforeStart(() => {
      this.use(router.middleware()); // middleware 等同 routes 
    });
    return router;
 }
```

这里我们用不到 router.ts 了, 所以我们只需要一个扩展一个阶段来获取这些注解数据, 然后重新注册即可

```js
// 伪代码演示
function newLoadRouter(){
	
    routers = [];
    // 获取全部类的注解. 
    routerPath = @Controller(routerPath)
    paramPath = @Get(paramPath)
    
    router = new Router({ sensitive: true, prefix: routerPath + paramPath }, this);
    routers.push(router)
    
  
    this.app.beforeStart(() => {   
        routers.forEach(router =>{
      		this.use(router.middleware()); // middleware 等同 routes 
        })
   	});

}
```













