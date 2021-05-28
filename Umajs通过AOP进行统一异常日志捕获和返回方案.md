## 背景

> 在编写接口路由时，在controller进行参数接受和拼接处理后，我们会调用service进行远程数据获取，而远程数据获取可能会因为网络，后端服务，参数变更等因素导致service调用错误。而此时我们为了记录对应接口的错误日志和记录，则需要对service通过try catch或者调用promise.catch进行捕获。

```ts
  @Path('/:id')
  async info(@Param('id') id: string) {
     let data :any;
    // 方案1
     try{
         data = await this.service.getInfo(id);
      }catch(err){
         this.logger.error(`获取活动数据失败${err}`)
      }
      
    // 方案2
     data = await this.service.getInfo(id).catch((err)=>{
           this.logger.error(`获取活动数据失败${err}`)
    });
      return Result.json(data);
  }
```
## AOP中aspect.afterThrowing进行错误捕获

此方案比较灵活，但存在的问题在于需要业务在调用的每一个method中都需要try 或者catch调用。而Umajs提供了AOP机制，我们可以统一aspect.afterThrowing进行method内异常捕获。

```ts
@Aspect.afterThrowing('err')
@Path('/:id')
  async info(@Param('id') id: string) {
     let data = await this.service.getInfo(id);
      return Result.json(data);
  }

// src/aspect/err.aspect.ts
export default class Err implements IAspect {
    afterThrowing(err: Error) {
        // 处理错误
        console.error(`uma接口路由发生错误：${err}`);
    }
}
```
通过`Aspect.afterThrowing`我们可以方便对method进行统一的捕获和日志记录。但如果业务想要知道个性化的错误返回和业务错误日志记录则无法做到了。比如：在afterThrowing中统一返回错误JSON给前台，打印 xxx接口调用异常等精确业务场景等日志信息。



## 插件统一处理错误

除了`aspect.afterThrowing`进行统一异常捕获之外，umajs插件市场中也提供了[@umajs/plugin-status](https://umajs.gitee.io/%E6%8F%92%E4%BB%B6/Status.html#%E9%85%8D%E7%BD%AE)可以进行统一的错误日志和统一返回。

```
import { IContext } from '@umajs/core';

export default {
    // eslint-disable-next-line no-underscore-dangle
    _error(error:Error, ctx: IContext) {
        ctx.status = 500;
				// 统一返回处理
        return ctx.view('error.html', { message: error.stack });
    },
};
```

但插件的方案也有限制，我们无法方便进行个性化错误信息的返回。如果要实现则需要通过ctx把业务日志传递下去。这并不是一个很好的解决方案。



## 解决方案

- afterThrowing接受自定义参数
- afterThrowing允许返回Result统一返回结果



# 效果

```
@Aspect.afterThrowing('err','info中获取服务端接口异常')
@Path('/:id')
  async info(@Param('id') id: string) {
     let data = await this.service.getInfo(id);
      return Result.json(data);
  }

// src/aspect/err.aspect.ts
export default class Err implements IAspect {
    afterThrowing(err: Error，data:any) {
        // 处理错误
        console.log(data,err);
        // 统一返回
        return Result.json({code:-1,message:data})
    }
}
```



注：以上实现还未发布，如果有更好的实现和处理方案，欢迎大家讨论提供建议。如果方案通过则会提交相关pr发布。



service异常如何处理，统一处理？

