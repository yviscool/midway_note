# 做一个 midway


## 序言

假设你熟悉 egg , egg 的 load 机制. 以及 egg-router 原理.  那么通过前面几节我们 injection 的剖析, 对照 midway.  我们要实现一个 midway, 怎么办?



## 前置知识

egg load 的机制

egg-router / koa-router 原理

injection 注入依赖 原理



egg 的 load 各个阶段 (强烈建议大致看过源码的阅读以下内容)

```js
'use strict';

const EggLoader = require('egg-core').EggLoader;


class AppWorkerLoader extends EggLoader {


  loadConfig() {
    this.loadPlugin();
    super.loadConfig();
  }


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

有 config , logger , plugin, service,  router.  看过 egg 源码的朋友应该都了解, egg 是通过 EggLoader 去框架, 应用, 插件目录下面去 loader 这些信息, 绝大部分最终都会挂载到 ctx/ctx.app = app上面. 反正我们写 egg 的时候, 有个 ctx 基本就无敌天上地下了.

如果你想要在 egg 之上封装一个上层框架, 就免不了跟这个 EggLoader 打交道. 而且你如果想要复用 egg 的生态而且要兼容, 这些 loadXXXX 绝大部分都是动不了的, 很多都是深度绑定的(第三方插件都是依赖这些约束去扩展的). 那么我们能做的只有扩展了.  

插个话题, 笔者之前看 nest 之后, egg 发布了 2.0. 我就想要把 nest 的一些特性移植到 egg 身上. 在看到 EggLoader 的时候, 大概就有了一些想法. 最终形成了一个 egg-pig 插件. 

```js
// app.js

const { EggLoader } = require('egg-core');

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

可以看到主要是利用 loadCustomApp 阶段加载 app.js 的时候, 暴力修改了原型链上的 loadController ,  然后扩展了原型链的一些东西.  这样以比较低的成本新增了许多特性同时兼容了 egg. 

回到主题上,首先 midway 是 egg 的扩展, 利用 ioc 的能力, 我们要实现几块解耦.

一块是和 egg 本身解耦, 使用装饰器完成各种 web 层的能力.

* 通过 @controller 和 @get @post 与 router 文件解耦

- 通过 @config 能力，和 app.config 解耦
- 通过 @plugin，和 app.xxx 插件解耦
- 通过 @inject() ctx 和请求链路解耦
-  .....

我们要实现这些功能, 势必就要继承这个 EggLoader 然后自定义 load 方法, 扩展一些方法. 

在进行这一块工作之前, 我们再来复习一个 injection. 前两章我们说到完成依赖注入有两个步骤, 

```js
const container = new Container();
container.bind(A);

async() => {
    await container.getAsync('A')
}
```

要先使用 container 对象 bind 生成 objectDefinition  对象. 最后使用 getAsync 来得到我们想要的对象. 那么有个问题, bind 的对象从哪来的. 我们哪些对象需要 bind ? 你会发现几乎所有使用装饰器的对象 service middleware controller 都需要 bind .  了解 EggLoader 肯定想到了, 我们可以扩展 loadController, loadService, loadMiddleware来做这件事.

显然这不是一种好的做法,  成本太大, 会被局限目录结构里面. 所以我们需要一种能自动扫描的能力, 不需要考虑目录结构. 那么怎么办呢. EggLoader 里面有一个 file_loader 里面有使用到 globby 模块. 这个模块大家或多或少都接触过,  就是根据规则扫描特定文件夹下所有文件的. 在 EggLoader 里面用来加载controller, service .... 这些特定目录的. 

所以说我们需要扩展一个阶段, 用 globby 扫描我们的主目录, 然后用 container 绑定这些 require('xxx')

```js
// 伪代码演示

this.loadApplicationContext(); // 我们扩展的阶段

// 扫描 baseDir 下面 ts js 文件, 然后自动绑定. 
function loadApplicationContext() {
    this.containerLoader = new Container();
      
    
      const fileResults = globby.sync(['**/**.ts', '**/**.js'], {
        cwd: baseDir,
        ignore: [
          '**/node_modules/**',
          '**/logs/**',
          '**/run/**',
          '**/public/**',
          '**/view/**',
          '**/views/**'
        ]),
      });

      for (const name of fileResults) {
        const file = path.join(dir, name);
        const exports = require(file);
        this.containerLoader.bind(exports);
      }
}
```

这里面还有一些细节,  当然这些细节不重要, 和我们主线无关. 我们所需要知道的就是怎么 bind 的.

**这样我们就完成了目录的解耦.**

在做完 IoC 自扫描能力之后，已经完全不需要考虑目录结构了，如果还需要 egg 的插件能力，目录还需要保留，如果不需要插件，就可以自由定义目录，扫描能力会完成一切。



然后看看我们最熟悉的路由是怎么解耦的. 我们通过 @Controller @Get @Post 这些来和 router.ts 解耦

```js
// router.ts
export default (app: Application) => {
  const { controller, router } = app;

  // router.get('/', controller.home.index);
};
```

在 loadRouter 阶段, 我们会通过 router.verb 来注册这些路由.  用过 koa-router 的都知道最终是由app.use(router.routes()) 来完成的. 那这个代码在哪里呢.

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

如果我们通过装饰器来控制, 这里就用不到 router.ts 了, 所以我们只需要一个扩展一个阶段来获取这些注解数据, 然后重新注册即可.

```js
// 伪代码演示
function newLoadRouter(){
	
    routers = [];
    // 获取所有注解为 CONTROLLER_KEY 的类, 就是我们的 Controller 类
    const controllerModules = listModule(CONTROLLER_KEY);
    
   
    // 注册 router
    for (const target of controllerModules){
        	// 获取 controller 这些 path 
         	const routerPath = getClassMetadata(CONTROLLER_KEY, target)
        	
            router = new Router({ sensitive: true, prefix: routerPath }, this);
        
        	// 获取 @Get @Post 这些 path 数据.
            const webRouterInfo = getClassMetadata(WEB_ROUTER_KEY, target);
        	
        	// 根据 webRouterInfo 生成 methodCallback, 然后用 router 注册
            methodCallback = generateController(webRouterInfo);
        	router.verb(paramPath, methodCallback)
        
   		    routers.push(router)
    }

  

    
  	// 最后 use
    this.app.beforeStart(() => {   
        routers.forEach(router =>{
      		this.use(router.middleware()); // middleware 等同 routes 
        })
   	});

}
```

这样我们就完成路由的解耦. 

当然这里面不止路由.  里面还有很多工作没做 @cofig @logger @plugin 以及一个很神奇的东西 @inject('ctx') 这些是怎么实现的.

```js
 public generateController(controllerMapping: string, routeArgsInfo?: RouterParamValue[]): Middleware {
    const [controllerId, methodName] = controllerMapping.split('.');
    return async (ctx, next) => {
      const args = [ctx, next];
      if (Array.isArray(routeArgsInfo)) {
        await Promise.all(routeArgsInfo.map(async ({ index, extractValue }) => {
          args[index] = await extractValue(ctx, next);
        }));
      }
      const controller = await ctx.requestContext.getAsync(controllerId);
      return controller[methodName].apply(controller, args);
    };
  }
```

todo 





