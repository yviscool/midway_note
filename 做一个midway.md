# 做一个 midway


## 序言

假设你熟悉 egg , egg 的 load 机制. 以及 egg-router 原理.  那么通过前面几节我们 injection 的剖析, 对照 midway.  我们要实现一个 midway, 怎么办?



## 前置知识

egg load 的机制

egg-router / koa-router 原理

injection 注入依赖 原理



首先 midway 是 egg 的扩展, 利用 ioc 的能力, 我们要实现两块解耦.

 一块是原本的插件, 配置, 上下文部分要融入到这个体系中, 实现对应的解耦.

- 通过 @config 能力，和 app.config 解耦
- 通过 @plugin，和 app.xxx 插件解耦
- 通过 @inject() ctx 和请求链路解耦

一块是目录结构的解耦,  在做完 IoC 自扫描能力之后，已经完全不需要考虑目录结构了.



egg 的 load 各个阶段

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



todo ...