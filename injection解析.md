# injection 解析


## 序言

injection 是 midway 的核心,  通过它来做管理依赖注入, 看源码时候也是重中之重, 后面 midway 都是围绕他结合 egg 扩展了一个 midway 出来, 如果无法理解他, 下面的东西就无法讲起, 在对 injection 进行解析之前, 各位应该先对 injection 有个基本的了解. [injection文档]([https://midwayjs.org/midway/ioc.html#%E6%B3%A8%E5%85%A5%E5%B7%B2%E6%9C%89%E5%AF%B9%E8%B1%A1](https://midwayjs.org/midway/ioc.html#注入已有对象))



## 前置知识

reflect-metadata , js 原型链

装饰器里面 reflect-metadata 是必须的, 用他来存取数据, 不懂的话可以看我之前 nest 笔记里面的解析

函数有一个属性叫做 name

xxxx.prototype 有个属性叫做 constructor 指向构造函数

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

这个初始化了什么的, 基本和上面的一样, 讲这个之前要和下面几个装饰器打交道

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

熟悉装饰器的朋友应该知道, 如果装饰器修饰普通属性是没有 index 属性的, 而且会把 propertyKey 传进来. 而修饰构造方法参数时候 index 会是参数索引值. propertyKey 会是 undefined

而 inject 修饰这两个功能是一样的, 都是以某个 key, 存储参数名的信息

```js

// 构造参数 
TAGGED  {
	0:  [ {key => 'inject', value => 'id(缺省为参数名)'} ]
}

普通属性

TAGGED_PROP => [
    propertyKey : [ {key => 'inject', value => 'id(缺省为propertyKey)'} ]
]

```

上面我们说过 inject 是基于 id 的注入. 在这里我们可以剧透一下, 具体 inject 的时候, 会根据 id 去对象工厂里面找对应的定义对象(下面会详细解释). 在这里我们只需要记住, inject 装饰器就保留了构造参数和普通属性的相关信息.

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

观察上面装饰器可以发现, 现在他们是毫无关联的. 而这个 bind 就是把它们关联在一起.  生成一个 ObjectDefinition 对象,  

```js
 {
			id: '指定的id或者(根据provide提供的驼峰或指定的id, 也就是上面TAGGED_CLS对应)'
			path: '类对象/构造函数'
			asynchronous: null,
			autowire: null,
			scope: 'singleton', // 默认为单例
			initMethod: null,
			destroyMethod: null,
			constructorMethod: null,
			export: null,
			dependson:[],
			// 构造参数
			constructorArgs : [{
				type: 'ref',
				name: 'id(缺省为 propertyKey )'
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

具体就是通过 reflect-metadata 根据上面的 key, 获取对应的值,  比如根据 TAGGED 获取生成构造参数 constructorArgs 相关的,  根据 TAGGED_PROP 获取生成  properties 相关的. 而 @aysnc() @init() @destory() @scope(scope) @autowire(isAutowire) 这几个我们说过了都是在 OBJ_DEF_CLS 整个对应的 key 里面. 这里面生成一个个属性, 比如 asynchronous 对应 isAsync, autowire 对应 autowire.

这里面比较困难的地方就是 properties.  实现的时候会查找原型链(也就是我们继承的时候), 遍历原型链上所有用 @Inject() 装饰的属性. 然后一起放到 innerConfig 里面.  比如 A 继承 B , 那么  Object.getPrototypeOf(O) === B, 我们就可以获取到 B, 然后就可以拿到 B 的一切. 无限遍历, 直到 等于 Function.prototype (一般来说, 有几种特殊情况, A 继承 null, undefined...). 



顺便一提的是 bind 可以手动 options 属性, 这个对应的是   

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



下面我们注入实例的时候, 全部都在这里面找的.





#### get 或者 getAsync 

这两个基本一毛一样, 只是将类实例化的时候, 某些方法能 await .

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
    
	bind (){}

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

我们来详细分析一下 getAsync,  首先 container 初始化的时候可以有一个 parent container. 这个有什么用我们以后, 我们以后结合 midway 来讲.

首先我们根据 id 去 registry 获取对应的 ObjectDefinition , 因为我们跟对象相关的属性全部在这里面. 上面为什么说 injection 注入说根据 id 注入的. 具体讲就是 map.get(id). 如果该对象不存在, 则去 parent 去找. 找不到抛出以后.

然后就是对象工厂根据 ObjectDefinition 来实例化了. 具体流程就是 

```js
// 如果是单例, 判断 singletonCache 有没有缓存, 如果有缓存,则命中
// 初始化依赖 也就是  definition.dependson []
// 初始化构造参数所需要的依赖
// 执行, beforeCreateHandler(this, clzz, constructorArgs, this.context), 
// 根据构造参数 创建对象
// 如果是请求作用域, 则绑定 egg , ctx 属性, value 为 this.context.get('ctx')
// 根据普通属性(创建依赖) 设置对象属性

// 自动注入
//  this.model = Autowire.createInject('fooModel') 这种形式
//  this.fooModel = null; 这种形式 也会被注入...

// 执行 afterCreateHandler (this, inst, this.context, defintion)
// 执行 init 方法
// 如果是单例作用域, 则 singletonCache 注册 该实例
// 如果是请求作用域, 则 registry 注册 registerObject 该实例
```



明天更新一些细节, 和补充说明. 坑有点多..........