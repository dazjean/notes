## Umajs会议内容记录

##  ssr

​	

- Umajs-react-ssr

  > 当前Umajs模板引擎采用koa-view插件可集成ejs,swig等传统社区nodejs模板引擎作为页面渲染，在前端开发团队页面开发模式当前更倾向于使用MVVM前后端分离框架，React,Vue  。相比较nextjs egg-react-ssr高度封装的解决方案，目标umajs-react-ssr仅仅提供一个支持支持jsx和服务端渲染能力的模板引擎。

  实现方案思路：

  - 不默认路由，路由均通过controller由业务决定。

  - MPA，页面组件之间默认代码隔离。
  - 支持React中预获取方式为：服务端获取和静态方法getInitalProps
  - 提供SSR与CSR两种模式动态切换
  - 支持SSR缓存模式
  - 自定义Html +SEO
  - 轻量级，不提供依赖客户端的API，方便老项目升级。

  ```ts
  this.reactView('index',initProps,{cache:true,ssr:true,useEngine:true})
  ```

最终实现[umajs-react-ssr](https://github.com/Umajs/umajs-react-ssr)

​	待办：

   - 自定义webpack配置方案

     > 当无法满足时提供webpack自定义配置，目前想法框架不提供特别的字段配置，1、通过更目录下自定义webpack.client.config.js  和 webpack.server.config.js，采用原生webpack配置去合并框架配置。2、配置文件统一扫描webpack.config.js。

-  升级webpack5.0

-  打包优化，服务端渲染bundle和客户端共用可行性

-  服务端渲染动态部署，web页面组件更改无需node重启

-  支持字体文件加载

-  ssr缓存模式文件缓存方案，当前按照URL 可能会导致服务器硬盘资源占用过大。应该按照path

## 插件

> 在umajs插件配置中，通过配置支持egg nest 生态的插件。

```ts
export default {
    'xxx': {
        enable: true,
        name: 'xxx',
        type:'egg',
        packageName: '@egg/xxx',
        options: {
            foo: '1'
        } // 插件的实际配置
    }
}
```



## class校验

> 参数修饰器基于[class-validator](https://github.com/Umajs/class-validator)提供@class修饰器支持。 

实现：https://github.com/Umajs/Umajs/tree/argDecorator-model/packages/arg-decorator#body%E5%8F%82%E6%95%B0%E8%A3%85%E9%A5%B0%E5%99%A8%E6%94%AF%E6%8C%81model%E6%A8%A1%E5%BC%8F



## 文档集成

>  类似https://github.com/nestjs/nest/blob/master/sample/11-swagger/src/cats/cats.controller.ts



## Umajs V2

https://github.com/Umajs/Umajs/compare/v2







