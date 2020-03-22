# injection 补充



injection 还有一个比较重要 DecoratorManager 这个东西. 顾名思义就是装饰器管理器, 这个其实是给第三方框架的用的. 和依赖注入本身无多大关系.

上一章我们讲了许多的装饰器以及 ioc 容器是如何处理类的依赖和类的初始化的. 但是这些仅仅能完成只是依赖注入这些功能.

```ts
import { provide, controller, inject, get } from 'midway';

@provide()
@controller('/user')
export class UserController {

  @config('hello')
  config;   

  @logger('myLogger')
  logger;

  @plugin()
  jwt;

  @get('/:id')
  async getUser(): Promise<void> {

  }
}
```



在 midway 里面 还有诸如 @controller  @config @logger @plugin @get @post .. 这些装饰器. 而这些装饰器都要保存各自的信息, 比如 @controller 的路由信息, @get @post 这些路径信息, @config @logger @plugin 这些要保存自己配置信息. 

所以说需要统一的容器来管理. 这个容器就是 DecoratorManager . 具体各位可以去翻一翻源码. 比较简单, 也是一个 map , 根据 key 保存这些数据, 源码核心也是 reflect-metadata. 

至于为什么这一层要交给 injection 来做, 而不是 midway 来做(实际上 midway 2.0 就直接把 injection 集成了). 一部分原因这个容器可以是通用的, 第三方纯碎的 web 框架都可以使用. 比如纯 express / koa . 

