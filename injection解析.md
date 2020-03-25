# injection 解析


## 序言

injection 是 midway 的核心,  通过它来做管理依赖注入, 看源码时候也是重中之重, 后面 midway 都是围绕他结合 egg 扩展了一个 midway 出来, 如果无法理解他, 下面的东西就无法讲起, 在对 injection 进行解析之前, 各位应该先对 injection 有个基本的了解. [injection文档]([https://midwayjs.org/midway/ioc.html#%E6%B3%A8%E5%85%A5%E5%B7%B2%E6%9C%89%E5%AF%B9%E8%B1%A1](https://midwayjs.org/midway/ioc.html#注入已有对象))



## 前置知识

reflect-metadata , js 原型链

装饰器里面 reflect-metadata 是必须的, 用他来存取数据, 不懂的话可以看我之前 nest 笔记里面的解析

函数有一个属性叫做 name

xxxx.prototype 有个属性叫做 constructor 指向构造函数

继承的时候 `__proto__` 和 `prototype` 指向

****



### provide(?id)

在对源码进行解析之前, 我们要记得一句话 **injection 是基于 id  的注入**. 下面会多次体现出来.

```ts
function provide(identifier?: ObjectIdentifier) {
  return function (target: any) {
     .....

    if (!identifier) {
      identifier = camelCase(target.name);
    }

    Reflect.defineMetadata(TAGGED_CLS, {
      id: identifier,
      originName: target.name,
    } as TagClsMetadata, target);

    // init property here
    initOrGetObjectDefProps(target);

    return target;
  };
}
```

provide 装饰器就干了两件事,   第一件以`TAGGED_CLS`为 key, 存储了

```js
TAGGED_CLS : {
      id: identifier, // 缺省值默认为函数驼峰名 (我们都知道 es6 的 class 是函数的语法糖)
      originName: target.name,
}
```

第二件就是

```js
// init property here
initOrGetObjectDefProps(target);
```

这个初始化了什么呢? 基本和上面的一样, 讲这个之前要和下面几个装饰器打交道

#### @aysnc() @init() @destory() @scope(scope) @autowire(isAutowire)

 这几个装饰器分别使用在对象, 对象方法上面, 功能是一样的, 以`OBJ_DEF_CLS` 为 key, 存储了装饰器对应的某个属性, 比如

``` 
@async() => {  isAsync: true }
@init() => { initMethod: propertyKey } 用在对象方法上面,  reflect-metadata 会把方法名(也就是 propertyKey) 传进来
@destory() =>  { destroyMethod: propertyKey } 同上
@scope(scope= ScopeEnum.Singleton) => { scope: scope } 默认是单例
@autowire(isAutowire=true) 
```

这几个装饰器都是干吗的, 我们下面再讲,现在只需要知道, 他们都是设置了某个属性.

那么回到上面的 initOrGetObjectDefProps(target) 

他就是 以`OBJ_DEF_CLS` 为 key, 值为 {}. 为后面的这些装饰器做了个前置工作, 后面的这些装饰器属性都在这个 {} 对象了里面.



#### 总结

```js
provide(id?) 
{
	OBJ_DEF_CLS => {  后面的这些装饰器属性都设置在这里
		isAsync: true => @aysnc()
		initMethod: propertyKey => @init()
		destroyMethod: propertyKey => @destory()
		scope: scope => @scope(scope)
		isAutowire: true =>  @autowire(true/false)
	},
	TAGGED_CLS => {
		id: id,
		originName: target.name
	}
}
```



### inject(?id)

上面的 provide 这个名字其实取的不好, 根据他的名字很难知道他是干嘛的. 

而 inject 这个装饰器我们清楚很多, 使用这个装饰器, 他对应的属性就会自动被注入, 让我们看看他到底干了啥

```ts
@provide()
@controller('/test')
export class TestController {

  @inject('fooService')
  service: FooService;

  constructor(
     @inject() barService: BarService
  ){
        
  }
}
```

inject 可以用在两个地方, 对象属性和构造参数上面.  

```ts
function inject(identifier?: ObjectIdentifier) {
  return function (target: any, propertyKey: string, index?: number): void {

  };
}
```

熟悉装饰器的朋友应该知道, 如果装饰器修饰普通属性是没有 index 属性的, 而且会把 propertyKey (也就是参数名)传进来. 而修饰构造方法参数时候 index 会是参数索引值. propertyKey 会是 undefined

而 inject 修饰这两个功能是一样的, 都是以某个 key, 存储参数名的信息

```js

// 构造参数 
TAGGED  {
	0:  [ {key => 'inject', value => 'id(缺省为参数名)'} ]
}

//普通属性

TAGGED_PROP => [
    propertyKey : [ {key => 'inject', value => 'id(缺省为propertyKey)'} ]
]

```

上面我们说过 inject 是基于 id 的注入. 在这里我们可以剧透一下, 具体通过容器实例化的时候, 会根据 id 去对象工厂里面找对应的定义对象(下面会详细解释). 在这里我们只需要记住, inject 装饰器就只保留了构造参数和普通属性的相关信息.

眼尖的朋友可能注意到了, 修饰构造参数的缺省为参数名,也就是说下面这个 value 应该是 barService. 

```js
constructor(
     @inject() barService: BarService
  ){
        
}
```

 而装饰器给我们的 propertyKey 只是 undefined. 那 injection 是怎么做得到的?

可以思考看看, 

可以思考看看, 

可以思考看看, 

可以思考看看, 

可以思考看看, 

答案就是用正则, 我们知道函数有个 toString 方法. injection 就是用正则匹配出来.



### container 容器

所谓的容器就是一个对象池，它会在应用初始化的时候自动处理类的依赖，并将类进行实例化。

#### bind(id, target, options)

观察上面装饰器可以发现, 现在他们是毫无关联的. 而这个 bind 就是把它们关联在一起.  生成一个 ObjectDefinition 对象,  如下

```js
 {
	id: '指定的id或者(根据provide提供的驼峰或指定的id, 也就是上面TAGGED_CLS对应)'
	path: '类对象/构造函数'
	asynchronous: false, // 对应上面的 @async()
	autowire: null,   // 对应上面的 @autowire(isAutowire)
	scope: 'singleton', // 默认为单例 对应上面的 @scope(scope)
	initMethod: null,  // 对应上面的 @init()
	destroyMethod: null,  // 对应上面的 @destory()
	constructorMethod: null,
	export: null,
	dependson:[],
	// 构造参数
	constructorArgs : [{
		type: 'ref',
		name: 'id(缺省为参数名)'
	}],
    // 普通属性
	properties: {
		innerConfig : {
			'参数名' =>  {
				type:'ref',
				name:'id(缺省为参数名)'
			}
		}
	}
}
```

具体就是通过 reflect-metadata 根据上面的 key, 获取对应的值.

比如根据`TAGGED`获取生成构造参数 constructorArgs 相关的, 根据`TAGGED_PROP`生成  properties 相关的. 

而 @aysnc() @init() @destory() @scope(scope) @autowire(isAutowire) 这几个我们说过了都是在`OBJ_DEF_CLS`这个对应的 key 里面. 这里面生成一个个属性, 比如 asynchronous 对应 isAsync, autowire 对应 autowire.

这里面比较困难的地方就是 properties.  实现的时候会查找原型链(也就是我们继承的时候), 遍历原型链上所有用 @Inject() 装饰的属性. 然后一起放到 innerConfig 里面.  比如 A 继承 B , 那么  Object.getPrototypeOf(O) === B, 我们就可以获取到 B, 然后就可以拿到 B 的一切. 无限遍历, 直到 等于 Function.prototype (一般来说这样的,但 源码有几种特殊情况,A 继承 null, undefined...). 



顺便一提的是 bind 可以手动 options 属性, 这个属性是   

```ts
export interface ObjectDefinitionOptions {
  isAsync?: boolean;
  initMethod?: string;
  destroyMethod?: string;
  scope?: Scope;
  constructorArgs?: IManagedInstance[];
  // 是否自动装配
  isAutowire?: boolean;
}
```

这个默认属性是可以被上面的  @aysnc() @init() @destory() @scope(scope) @autowire(isAutowire) 覆盖的.



上面的 ObjectDefinition  这个对象是放在一个 map 里面的, 具体就是 ObjectDefinitionRegistry , 这个对象扩展了 map . 

```js
this.registry.registerDefinition(id, definition);
```

我们来看看这个 ObjectDefinitionRegistry  长什么样

```js
registry extends Map => {
	singletonIds: [], 

	registerDefinition(id, definition){
		如果 definition 是单例 则 singletonIds push 该 id,
		this.set(id, definition)
	}
	registerObject(){
		this.set('id_default_' + id, target)
	}
}
```



下面我们注入实例的时候, 全部都在这个registry 里面找的. 这也就是为什么 id 那么重要了.





#### get 或者 getAsync 

这两个基本一毛一样, 具体区别下面讲

我们来看看 container 这个对象有什么

```ts
container {
    
   constructor(baseDir = '', parent?) {
     this.parent = parent;
     this.baseDir = baseDir;

     this.init();
   }

	registry extends Map {
		上面我们 bind 的对象, 全部在这里面
	}

	managedResolverFactory {
	
	}
    
	bind (){
        // 关联所有装饰器, 生成一个 ObjectDefinition 对象
    }

    async getAsync<T>(identifier: ObjectIdentifier, args?: any): Promise<T> {

      if (this.registry.hasObject(identifier)) {
        return this.registry.getObject(identifier);
      }

      const definition = this.registry.getDefinition(identifier);
      if (!definition && this.parent) {
        return this.parent.getAsync<T>(identifier, args);
      }

      if (!definition) {
        throw new NotFoundError(identifier);
      }

      return this.getManagedResolverFactory().createAsync(definition, args);
    }


}
```

我们来详细分析一下 getAsync / get ,  这里面第一个差异就是 @async() 这个装饰器, get 里面会先检查是否是异步创建的, 如果为真则抛出异常, 强制要求我们用 getAsync . (@async已经被废弃, 不推荐使用, midway 全部用的都是 getAsync)

```js
  get<T>(identifier: ObjectIdentifier, args?: any): T {
	if (this.registry.hasObject(identifier)) {
       return this.registry.getObject(identifier);
    }

    if (this.isAsync(identifier)) {
      throw new Error(`${identifier} must use getAsync`);
    }

    const definition = this.registry.getDefinition(identifier);
    if (!definition && this.parent) {
      if (this.parent.isAsync(identifier)) {
        throw new Error(`${identifier} must use getAsync`);
      }

     ...
    }
	...
  }
```



首先 container 初始化的时候可以有一个 parent container. 这个有什么用我们以后讲.

第一步如果 registry.registerObject 过该对象, 则直接返回. (这个用来获取一些工具类, 或者框架常用属性配置对象其他等等). 比如官网的例子

```js
// in global file
import * as urllib from 'urllib';
container.registerobject('httpclient', urllib);

@provide()
@controller()
class A{
    
    @inject()
    httpclient;
    
}
```

接着我们根据 id 去 registry 获取对应的 ObjectDefinition , 因为我们跟对象相关的属性全部在这里面. 

上面为什么说 injection 注入说根据 id 注入的. 具体讲就是 map.get(id). 如果该对象不存在, 则去 parent container 去找. 找不到抛出异常.

然后就是对象工厂根据 ObjectDefinition 来实例化了. 具体流程就是 



* 如果该 definition是单例作用域( @scope( ScopeEnum.Singleton)), 判断 singletonCache是否有缓存, 如果有命中,返回实例对象. (这里的 singletonCache 也是一个 map, 专门用来存放单例对象的)



* 初始化依赖 也就是 definition.dependson,(这个没卵用, injection 没看到任何可以赋值的地方, midway 也没用到)

```js
  // 预先初始化依赖
    if (definition.hasDependsOn()) {
      for (const dep of definition.dependsOn) {
        await this.context.getAsync(dep, args);
      }
    }
```

* 取出上面的 definition.path 对象.  一般就是 class 对象

```js
const Clzz = definition.creator.load();
```

```js
  load(): any {
    let Clzz = null;
      //最早 injection 是使用 xml 注入的, 这里的 path 会是 string.  而且 definition.export 也就在这里面有用. 了解一下就行.
    if (typeof this.definition.path === 'string') {
      // 解析xml结果 默认 path = '' 需要兼容处理掉
      if (!this.definition.path) {
        return Clzz;
      }
      const path = this.definition.path;
      if (this.definition.export) {
        Clzz = require(path)[this.definition.export];
      } else {
        Clzz = require(path);
      }
    } else {
      // if it is class and return direct
      Clzz = this.definition.path;
    }
    return Clzz;
  }
```



* 取出 definition.constructorArgs 相关信息, 对构造参数注入依赖, 也是重复这个流程

```ts
const Clzz = definition.creator.load();
let constructorArgs;
if (definition.constructorArgs) {
	constructorArgs = [];
	for (const arg of definition.constructorArgs) {
	  constructorArgs.push(await this.resolveManagedAsync(arg));
	}
}
```



* 执行前置钩子 beforeCreateHandler(this, clzz, constructorArgs, this.context) 

```js
beforeCreateHandler = [];
beforeEachCreated(fn) {
    this.beforeCreateHandler.push(fn);
}
```

钩子就是注册的一个函数, 在合适的时机调用传值而已, (midway 里面会用到, 可以先想想干吗用的)

* 根据构造参数和 clzz 对象实例化对象. 

```js
let inst = await definition.creator.doConstructAsync(Clzz, constructorArgs);

...
async doConstructAsync(Clzz: any, args?: any): Promise<any> {
    let inst;
    // 如果 definition 有指定 constructMethod 方法, 则用该方法实例化, (一般用不到)
    if (this.definition.constructMethod) {
      const fn = Clzz[this.definition.constructMethod];
        inst = fn.apply(Clzz, args);
    } else {
      inst = Reflect.construct(Clzz, args); // 等同 new Object(...args)
    }
    return inst;
  }
```

* 如果是 definition 是请求作用域, 那么给 inst 赋值一个 _req_ctx 属性. 

```js
Object.defineProperty(inst, '_req_ctx', {
        value: this.context.get('ctx'),
        writable: false,
        enumerable: false,
 });
```

* 取出 definition.properties 相关信息, 对普通注入依赖(也是重复这个流程), 然后给对象设置属性. 

```js
inst[ key ] = this.resolveManaged(identifier);
```



* 自动注入过程, 也就是上面的 @autowire(), 

```js
if (definition.isAutowire()) {
      Autowire.patchInject(inst, this.context);
      Autowire.patchNoDollar(inst, this.context);
 }
```

这个是干什么的呢,  也就是下面这两种形式也会根据 id 被注入

```js
constructor(){
    //  this.model = Autowire.createInject('fooModel') 这种形式也会被注入, 不推荐使用
    //  this.fooModel = null; 这种形式也会被注入, id 就是 fooModel  不推荐使用
}
```

* 执行后置钩子 afterCreateHandler(this, inst, this.context, defintion) (midway 里面会用到, 可以先想想干吗用的)

```js
afterCreateHandler = [];
afterEachCreated(fn) {
    this.afterCreateHandler.push(fn);
}
// 同上
```

* 执行 @init() 所绑定的方法, 根据上面的流程我们可以知道所有属性都已经注入完了. (就可以自由发挥了)
* 如果是单例作用域, 则 singletonCache 注册该实例
* 如果是请求作用域, 则 registry 注册 registerObject 该实例
* 实例化对象完成, 返回该对象



总结 container.get(id) 就是根据 id 取出  definition 对象, 拿出相关信息, 实例化对象.

其中可能让人比较困惑就是 scope 作用域,  钩子函数. 

#### scope 作用域

scope 的作用域有三种

```ts
export enum ScopeEnum {
  Singleton = 'Singleton', // injection 默认是单例
  Request = 'Request',  // midway  默认是 请求作用域
  Prototype = 'Prototype',
}
```

通过流程我们知道, 所谓的作用域就是一个标记.

单例标记能体现的只有 singletonCache 这个 map , 如果工厂实例化的时候有缓存则取出来没有则在实例化结束的时候存储起来.

请求标记能体现的就是 registry 这个 map, 这个 map 也是放我们 definition 对象, 只不过请求标记有一个前缀不一样(下面的 registerObject),  和单例标记一样如果工厂实例化的时候有缓存则取出来没有则在实例化结束的时候存储起来.

```js
registry extends Map => {
	singletonIds: [], 

	registerDefinition(id, definition){
		如果 definition 是单例 则 singletonIds push 该 id,
		this.set(id, definition)
	}
	registerObject(){
		this.set('id_default_' + id, target)
	}
}
```

而原型作用域通过我们上面流程所知, 是没有缓存的, 所以每次 get 的时候都是新的对象.

那么乍一看, 单例标记和请求标记好像没有什么区别. 别着急, 这一点我们得和 parent container 和 egg 的 context 结合到一起在看的出来. 

#### 钩子函数

按照上面的流程, 钩子函数分两种, 一种前置, 一种后置, 而且执行的位置也比较巧妙.

**前置钩子是在我们 new 之前做的. 而后置钩子是在 autowire 之后也就是属性全部注入完成之后执行的.**

```js
// 前置钩子
for (const handler of this.beforeCreateHandler) {
  handler.call(this, Clzz, constructorArgs, this.context);
}
.....
.....

// 后置钩子
for (const handler of this.afterCreateHandler) {
  handler.call(this, inst, this.context, definition);
}
```

还记得上面的 new 流程吗, 根据构造参数和 clzz 对象实例化对象. 

```js
async doConstructAsync(Clzz: any, args?: any): Promise<any> {
    let inst;
    inst = Reflect.construct(Clzz, args);
    return inst;
}
```

而前置钩子会接受构造参数 constructorArgs , 这意味着 constructorArgs  可能会发生改变. 

聪明的朋友肯定想到了, 这里就先不剧透了.

