## 背景

目前团队的老项目基于express,经过长年需求的迭代维护，业务逻辑非常复杂，维护和定位问题非常困难，而现在直接全部重构是不现实的，所以针对这样的老项目只能部分接口开始逐步重构。而笔者团队作为umajs的拥护者，当然优先考虑的是如何基于umajs进行重构express的项目了。首先简单罗列一下我们目前存在的一些问题吧，如果读者发现和自己负责的项目比较类似那么恭喜你，你有机会改变现状了。



## 存在的问题

1、项目中异步请求多个服务时地狱式回调。

2、代码没有注释，方法也没有类型参数定义，function一撸到底。难读，难维护。

3、工程目录没有规范，工具类方法散落四处，代码复用率低

4、代码异常捕获机制不完善，系统脆弱易崩

5、日志不规范，线上排查定位问题困难

6、接口，参数管理太乱，各种参数判断，类型转换；不易形成文档

.......



## 为什么要寻求兼容express老项目的解决方案

整体重构不行吗？当然是可以，你需要有充足的人力，对业务特别熟悉，业务也能给你充足的时间。遗憾的是我们没有这样的机会，业务还在不断迭代更新；所以我们需要在现有的代码中引入一种新的解决方案，同时老的代码还能正常运行,当有新的接口或者需求迭代时部分慢慢迁移到新方案中。也就是说已存在的路由请求不动，当有新的接口和对路由进行重构时使用umajs重写。

笔者负责的项目技术栈为：express原生路由+[yog2](http://fex.baidu.com/yog2/docs/)+ejs。而我们想要解决的是node服务端代码的问题，所以在改造的同时需要保证前端模板的渲染模式不变，而yog2基于swig自己做了一层封装，增加了一些过滤器和管道方法，所以需要在umajs中依然使用express项目中引入的模板引擎。而Umajs灵活强大的plugin机制可以方便我们在中间件，插件中将express的中间件和插件挂载到koa上下文中继续使用。



好了，我们来看下Umajs给出的解决方案是什么。

## 解决方案

### 方案1：基于Umajs做一个适配express的框架Umajs-express. 

#### 优点：

  - 支持Umajs所有的基础功能，使用时只需要把@umajs替换成umajs-express
  - 框架在中间件中封装了ctx上下文，在controller，service中通过this.ctx可以直接获取到ctx,保持了原汁原味的Umajs使用风格和习惯
  - Umajs-express版本跑过了100%Umajs的单元测试，后期可以无缝替换@umajs-express为@umajs

#### 缺点：

- 插件和中间件的实现方式和umajs不一样 ，中间件handler处理函数为原生express中间件。



#### 接入方式：

```ts
// app.ts
import * as express from 'express';
import Uma from '@umajs-express/core';
import { Router } from '@umajs-express/router';
import { TUmaOption } from '@umajs-express/core';

const options: TUmaOption = {
    Router,
    bodyParser: true,
    ROOT: __dirname,
    env: process.argv.indexOf('production') > -1 ? 'production' : 'development',
};

(async () => {
    if (process.argv.indexOf('--express') > -1) {
        const app = express();

        app.use(await Uma.middleware(options, app));

        app.listen(8058);
    } else {

        const uma = Uma.instance(options);

        uma.start(8058);
    }
})();

```

```ts
// index.controller.ts
import { BaseController, Path, Private, Param, Query, RequestMethod, Aspect, Service, Result } from '@umajs-express/core';
import { Get, Post } from '@umajs-express/path';

export default class Index extends BaseController {
    @Path('/home')
    home() {
        this.setHeader('clientType','PC');
        console.log(this.ctx.get('Cache-Control'));
        return Result.send('this is home router! '+this.getHeader('Cache-Control'));
    }
    @Get('/reg/:name*')
    @Aspect.around('mw')
    reg(@Param('name') name: string) {
        return Result.send(`this is reg router. ${name}`);
    }
}
```





### 方案2：基于Uma.callback ，采用中间件形式集成到express,connect框架中。【笔者采用】

#### 优点：

- Umajs获取到框架实例化后的koa.callback.可以集成到express框架中。
- 项目中直接升级到Koa,相比Umajs-express方案后续没有迁移成本，一步到位。

#### 缺点：

- 在express中使用koa时，koa的执行顺序需要在express所有中间件之后，如果之前项目有设置404路由需要删掉404中间件，通过koa重写。

#### 接入方式：

```ts
//app.ts
(async () => {
    const callback = <any> await Uma.callback(options);
    const app = express();
    app.use((req, res, next) => callback(req, res).then(() => {
        if (res.headersSent) return;
        next();
    }));
})();

```

#### 注意事项：

1、建议在umajs中间件中重置ctx.status=200状态码。

2、umajs中配置的view插件和express中使用的模板引擎保持一致.或者直接通过插件挂载到context和Result对象

3、express中存储的token和session通过umajs插件挂载到ctx中继续使用。



## 建议

- 如果团队技术氛围比较忠于express,建议采用方案1；可以获得umajs的开发体验的同时保持技术栈路线
- 如果希望后期整体升级为Koa，建议采用方案2。



---


Umajs 是58同城推出的一款轻量级 node web 框架。它的中文含义是大熊座，北斗七星都是它的组成部分；正如同 Umajs 也是由不同的 package 所组合在一起。我们希望 Umajs 的每一部分，都是优秀的、闪耀的、经受的住各种大型项目检验的。
欢迎大家start支持[umajs](https://github.com/wuba/Umajs)。
### 链接地址：

https://github.com/wuba/Umajs

https://github.com/Umajs/umajs-express

使用文档：

https://umajs.github.io/%E6%96%B0%E6%89%8B%E6%8C%87%E5%8D%97/%E6%A1%86%E6%9E%B6%E4%BB%8B%E7%BB%8D.html