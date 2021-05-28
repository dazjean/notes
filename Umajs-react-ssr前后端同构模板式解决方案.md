### Umajs框架前后端同构最佳实践

## 简介

> Server Slide Rendering，缩写为 **ssr** 即服务器端渲染，在这儿我们不讨论基于传统模板引擎实现的服务端渲染方案；目前业内前端流行的库和框架主要是React和Vue,MVVM类的开发模式和组件化开发思想提高了前端的生产力，使得构建一个web页面变得越来越容易；但也面临了新的问题，主要是首屏加载白屏，SEO两种场景。在业内的解决方案有Nextjs和Nuxtjs分布针对React和Vue技术栈提供了SSR的解决方案，而这两者虽然解决和降低了实现SSR的门槛，但要结合企业级node开发框架使用时，同构方案的集成和学习成本则并不友好。这也是我们基于Umajs进行react ssr进行探索的原因。

## 为什么要进行前后端同构

> 得益于node技术栈的繁荣和发展和SSR技术的出现，随着node越来越被更多的项目使用。我们发现nodejs不仅仅是作为SSR的工具,在node后端工程里面还作为接口网关以及做一些数据聚合的工作，通过node可以直接和后端接口，数据库进行交互获取数据。在这背景下，前端不仅仅需要维护前端工程，还需要维护一个node工程。这逐渐成为前端团队面临的新的痛点。

- SSR
- 前后端协作方式变化，node承担更多的数据聚合工作

## 现有方案存在的问题

> 当前最流行的SSR同构方案有nextjs,egg-react-ssr(先已经升级成ykfe/ssr),在Vue技术栈有Nuxtjs。通过调研对比发现这些框架或者解决方案存在如下的问题。

- 学习，升级成本较高，已有方案封装都比较重。

   页面组件开发需要引入额外框架特有的API和模块。比如SEO动态改写html标题和关键字，引入第三方SDK,页面跳转等。

- 环境区分复杂；

  当项目需要部署测试环境，沙箱环境，线上环境，前端需要额外进行环境区分适配各个环境。而且对于服务端渲染模式时，开发者要额外在代码中进行isSSR or isCSR这样的变量区分，以满足不同渲染场景下的数据获取方式。

- 页面路由不够灵活；

  基于文件页面组件目录路由的解决方案，默认页面组件和路由建立一套一一对应的映射规则。比如：`pages/list/index.tsx` 自动会映射成`/list`。而当需要对页面组件进行一个AB测实验时，就难以去灵活控制，框架势必需要通过额外的配置文件去完成一套新的路由映射规则。这对开发者来说又增加了学习成本。而且默认按照页面组件命名映射路由这种方式在前后端同构的项目中也存在路由维护的成本，开发者也需要注意区分页面组件路由和接口路由的命令冲突问题。

- 服务端渲染数据预获取形式受限

  目前已有方案大多采用静态方法获取服务端渲染初始化数据，具体实现方式在页面组件中暴露静态方法`getInitalProps`,在方法提中接受服务端请求上下文，然后通过调用支持浏览器和node环境请求http的工具请求接口返回数据。但这种方式有两个比较严重的问题；一是服务端和客户端请求方式不统一，开发者需要针对两种情况进行编写数据获取逻辑。二是难以进行服务端断点调试，三是对于非http请求获取数据方式不友好，目前也有的框架在支持非http方式时，会将服务端获取数据的函数挂载到请求上下文中。但这种方式时就会回到第一个问题，客户端渲染时则需要将服务端函数包装成一个http的接口去替代兼容。

  

而在58内部，自研企业级node web开发框架Umajs作为前端node项目基建框架，我们有大量内部技术和业务项目使用umajs搭建中间层和构建web页面，现有前后端同构+SSR的解决方案和Umajs框架都有着水土不服的情况，所以我们自研了Umajs-react-ssr的解决方案来完美搭配Umajs。

## umajs-react-ssr介绍

> 以React SSR为例，react-dom/server提供了renderToString以及renderToNodeStream方法可以将我们编写的页面组件在node中解析成html字符串或者stream。然后将返回的字符串注入挂载到客户端渲染时设置的root元素输出到浏览器，在客户端加载js,和css文件然后调用hydrate方法。这是所有SSR解决方案统一的流程和原理，也包括本方案，感兴趣的自行搜索下，业内已经有很多相关的原理介绍。

本文主要通过以下几个方面的使用来介绍umajs-react-ssr的特性。

- Pages
- 页面路由以及动态路由
- 服务端数据获取
- 应用配置
- SEO
- 状态管理

## 特性介绍

- 目录以及Pages

  | 方案            | 目录               | 前后端页面路由 |
  | --------------- | ------------------ | -------------- |
  | nextjs          | pages/xxx.ts       | /xxx           |
  | egg-react-ssr   | pages/xxx/index.ts | /xxx           |
  | Umajs-react-ssr | pages/xxx/index.ts | 不默认路由     |

  在`umajs-react-ssr`中pages目录下存放各个页面组件，比如`pages/list/index.ts`,这和我们日常开发MPA项目保持一致的规范，`list`文件夹下的index.ts会作为webpack打包的入口文件进行编译，在开发模式会被编译成提供给客户端和服务端渲染使用的bundle文件。相比较其他方案我们最大的不同是不默认提供和限制页面的路由，`list`页面会作为ssr模式的页面标识在`umajs`的`controller`中被使用。

  

  页面组件编写

  ```typescript
  import React from 'react';
  type typeProps = {
      ListData :[string]
  }
  export default function (props:typeProps){
       const ListData = ['itme1','itme2','itme3','itme4'];
      return (
          <div className="list" style={{textAlign: 'center'}}>
              <h3>列表</h3>
              <ul>
                  {ListData.map((item,value)=>{
                      return (
                          <li key={value}>
                                <div className="item">{item}</div>
                          </li>
                      )
                  })}
              </ul>
          </div>
      )
  }
  ```

  

-   页面路由以及动态路由

  - 页面路由

    > 传统SSR解决方案在编写page页面组件时，就已经决定了服务端访问页面路由时的url,比如：`pages/list/index.ts`会被自动映射到路由`localhost:port/list`。在umajs-react-ssr中我们可以更灵活的控制路由将要渲染具体哪一个页面组件，同时我们可以给一个页面组件映射多个路由，这在做AB测以及对页面进行大改版时会特别有帮助。

  ```typescript
  
  import { BaseController,Path , Query} from '@umajs/core';
  import { Result } from '@umajs/plugin-react-ssr'
  
  export default class Index extends BaseController {
      @Path("/","/list")
      index() {
          return Result.reactView('list');
      }
     @Path("/","/ABlist")
     index(@Query('abtest') abtest:string) {
        let viewName = 'list'+abtest;
        return Result.reactView(viewName);
      }
  }
  ```

  - 动态路由

  > 在`nextjs` 和`ykfe/ssr`提供解决方案中，动态路由根据页面组件入口文件的名称进行区分；比如：`pages/detail/[id].ts`或者`pages/detail/render$id.ts`。再或者类似`egg-react-ssr`中提供`config.ssr.js`配置文件去配置我们的路由和映射规则。而在`umajs-react-ssr`中我们使用更简单，因为我们的页面组件目录和路由没有关联，所以也不需要改变我们的文件命名，以及去熟悉又一个新的动态路由的规则。

  ```typescript
  
  import { BaseController,Path ,Param} from '@umajs/core';
  import { Result } from '@umajs/plugin-react-ssr'
  
  export default class Index extends BaseController {
      @Path("/detail:id")
      index(@Param('id') id:string) {
      		const detailData = this.detailService.get(id);
          return Result.reactView('detail',detailData);
      }
  }
  
  ```

  - 嵌套路由

  >  由于我们没有提供默认前端路由，所以我们不会有额外的router.js繁杂的配置，在使用React-router时保持前后端路由一致即可
  
  ```js
  //web/browserRouter/index.js
  import React, { Component } from 'react';
  import { Route, Switch } from 'react-router-dom';
  export default class APP extends Component {
      render() {
          return (
              <div className="demo">
                  <Switch>
                      <Route exact path="/" component={Home} />
                      <Route exact path="/about" component={About} />
                      <Route component={Home} />
                  </Switch>
              </div>
          );
      }
  }
  ```

  

  服务端Controller路由设置
  
  ```typescript
  import { BaseController,Path ,Parms} from '@umajs/core';
  import { Result } from '@umajs/plugin-react-ssr'
  
  export default class Index extends BaseController {
      @Path("/browserRouter","/browserRouter/:path")  //// 服务端路由和react-router路由规则保持一致
      browserRouter() {
        return Result.reactView('browserRouter');
      }
  }
  ```
  
  



- 数据获取

  > 数据获取是服务端渲染最重要的环节，在CSR模式下，数据通常是在生命周期`componentDidMount`中发起，然后调用setData触发页面渲染。SSR模式中有两种数据获取方案，静态方法获取以及服务端获取。

  - 静态方法getInitialProps

    > getInitialProps静态方法是由`nextjs`提出的概念，是在组件实例中挂载一个static静态方法，当服务渲染时预先调用此方法获取到数据，然后再SSR阶段通过props初始化到页面组件中，从而得到完整的html结果。这种方案也被业内其他框架追随，包括`egg-react-ssrs` `ykfe/ssr` `easy-team`等。

    ```js
    function Page(props) {
      return <div> {props.name} </div>
    }
    
    Page.getInitialProps = async (ctx) => {
      return Promise.resolve({
        name: 'Egg + React + SSR',
      })
    }
    
    export default Page
    ```

    上面样例来自`egg-react-ssr`，通过这种方式可以将服务端获取数据的逻辑和客户端页面渲染编写在一个文件中。在ssr时静态方法在node端被调用，csr时通过高价组件包装，在link或者API跳转时执行。而这种方式的实现其实在真实场景中并不友好，当前前后端的交互不一定是http,而是rpc框架或者直接从数据库中获取并加工获得。这种方式就存在很大的局限，一是不方便node调试，而是必须使用http作为前后端交互通信方式，而且这种将node环境和客户端页面组件混写在一起的方式也会增加开发人员的理解，所以我们也支持了在服务端获取数据的方式。

  - 服务端获取

    > `Umajs-react-ssr`页面组件的渲染是在`controlle`r路由中进行分发调用，在`controller`中我们可以直接调用`service`通过rpc或者数据库中获取到初始化的数据，同时进行加工并在指定渲染页面组件时传递给页面组件。在react中无论是`csr`还是`ssr`模式下我们直接通过`props`可以获取到服务端初始化的数据。这样让页面组件专注于视图的渲染，数据的处理和加工交给node来实现。在客户端也无需发起http请求，效率也会更高。

    ```typescript
    import { BaseController,Path } from '@umajs/core';
    import { Result } from '@umajs/plugin-react-ssr'
    
    export default class Index extends BaseController {
        @Path("/list")
        index() {
            const ListData = ['itme1','itme2','itme3','itme4','itme5','itme6','itme7','itme8']
            return Result.reactView('list',{title:'列表',ListData},{cache:true});
        }
    }
    ```

    

- 应用配置

  > 无论是nextjs还是业内其他号称最简单，最开箱即用的服务端渲染框架，都没有真的做到开箱即用；开发者还要学习众多各种规范的配置项，包括webpack配置，运行时配置，路由配置等等。

  

  

  

  而umajs-react-ssr的配置则是非常简单的，我们内部集成了webpack的常用配置和内置了`bable`,`js`,`ts`,`css`,`ur`l,`scss`,`less`各种常用loader。开发者初始化工程后除了在plugin中引入@umajs/plugin-react-ssr后无需再单独配置众多框架特有配置文件。

  ```typescript
   // src/config/plugin.config.ts
      export default <{ [key: string]: TPluginConfig }>{
          'react-ssr': {
              enable:true,
              options:{
                  rootDir:'web',// Pages目录根路径
                  ssr: true, // 全局开启服务端渲染
                  cache: false, // 全局使用服务端渲染缓存 开发环境设置true无效
              }
          }
      };
  ```

  没错，我们提供的配置项就是如此简单，通过插件配置我们提供了全局开启和关闭服务端渲染和缓存策略。在运行期，通过插件导出的`Result.reactView`对页面进行ssr时我们还可以进行动态调整。

  ​		- 关闭开启缓存	

  ```typescript
  export default class Index extends BaseController {
      @Path("/list")
      index() {
          const ListData = ['itme1','itme2','itme3','itme4','itme5','itme6','itme7','itme8']
          return Result.reactView('list',{title:'列表',ListData},{cache:true}); //服务端数据是固定不变的 开启缓存，优化首屏加载时间
      }
  }
  ```

  

  ​		- csr和ssr模式动态切换

  ```typescript
  export default class Index extends BaseController {
      @Path("/list")
      index() {
          const ListData = ['itme1','itme2','itme3','itme4','itme5','itme6','itme7','itme8']
          return Result.reactView('list',{title:'列表',ListData},{ssr:false}); //流量高峰时可通过AB测或者其他方案动态调整服务端渲染和客户端渲染流量占比
      }
  }
  ```

  

- SEO

  > NextJs以及其他解决方案采用的是`ALL in js`思想，将HTML head 里面 meta 信息也作为 React 服务端渲染的一部分，这很符合一切皆为组件的概念；但在使用起来却有一定的上手难度和学习成本，我们需要属性各种`Layout`的用法配置。

  ![image-20210508171451667](https://tva1.sinaimg.cn/large/008i3skNly1gqb54mscqoj30w90prn1f.jpg)

  ​																											egg-react-ssr实现

  



![image-20210508171855415](https://tva1.sinaimg.cn/large/008i3skNly1gqb548n8vyj30ir0g00u9.jpg)

​																			    NextJS实现方案

无论是egg还是Nextjs的高度包装实现对开发者来说都有一定的上手成本，对于大部分开发者来讲我们最熟悉的还是html,`ALL in JS`的使用还是应该有限度的,页面中的`head,viewport,title`甚至引入第三方`SDK`场景在`React`组件中实现且进行服务端渲染适配都会增加框架的复杂和规则的学习。React,或者Vue也不推荐这样做。所以我们保留和采用了传统html模板，通过`htmlwebpackplugin`结合`nunjucks`引擎来实现。在`Pages/xxx/index.tsx`同目录下，开发者可以定义`index.html文件`。使用方法完全遵循[htmlwebpackplugin](https://webpack.docschina.org/plugins/html-webpack-plugin/)的使用，同时我们可以在html中使用nunjucks的语法解析从controller中初始化到页面组件的数据。



 html模板

```html
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <meta charset="utf-8">
    <meta name="viewport" content="initial-scale=1.0,maximum-scale=1.0,minimum-scale=1.0,user-scalable=0,width=device-width">
    <meta name="format-detection" content="telephone=no">
    <meta name="format-detection" content="email=no">
    <meta name="format-detection" content="address=no;">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="default">
    <meta name="keywords" content="{{keywords}}">
    <title>{{title}}</title>
    <!-- 引入第三方外链样式文件 -->
</head>

<body>
    <div id="tempate">{{msg}}</div>
    <div id="app"></div>
      <!-- 引入第三方SDK -->
</body>
</html>
```



服务端数据注入

```typescript
// 服务端路由初始化页面数据时传递的数据都可以在html中通过nunjucks模板引擎进行渲染
export default class Index extends BaseController {
    @Path("/list")
    index() {
        const ListData = ['itme1','itme2','itme3','itme4','itme5','itme6','itme7','itme8']
        return Result.reactView('list',{title:'列表',keywords:'ssr,umajs-react-ssr'，ListData},{cache:true,useEngine:true}); 
    }
}
```

​																												Umajs-react-ssr实现方案



- webpack默认配置

  通常情况框架默认配置已经可以覆盖到日常的开发，关于一些没有覆盖到常用配置，可以提issue由框架来支持。如果开发者想个性化配置，umajs-react-ssr也提供了默认扫描的配置文件，在项目根目录路径下通过webpack.config.client.js 和 webpack.config.ssr.js的属性和框架内置配置进行合并。配置属性参考webpack 4.0官方文档。

  ```js
  module.exports = {
      module: {
              rules:[{
                      test: /\.(png|jpg|jpeg|gif|svg)$/,
                      use: [
                          {
                              loader: 'url-loader',
                              options: {
                                  name: '[hash:8].[name].[ext]',
                                  limit: 8192,
                                  outputPath: 'images/'
                              }
                          }
                      ]
                  }]
             },
      alias: {
          images: path.join(process.cwd() + '/web/images'),
          components: path.join(process.cwd() + '/web/components')
      },
   };
  ```

  ​															   更容易,更灵活个性化配置

  

## 总结

> 相比较业内实现umajs-react-ssr解决方案对开发者最友好，对页面组件开发和视图没有额外的限制，我们也更加灵活控制路由，将路由控制权交给后端来控制，使得前端部分更专注视图的渲染。通过和其他方案的对比我们也总结umajs-react-ssr具有以下的优点，也为开发者提供选择和参考。

- 不默认路由，不需区分前端路由和后端路由概念，且支持页面级组件AB测；`灵活`

- 页面组件中没有`__isBrowser__`之类变量对`ssr`和`csr`模式进行特殊区分处理;`统一`。

-  自定义`HTML`采用`htmlWebpackPlugin`,没有`runtime`，页面响应速度更高;`高性能`。

- 支持`html`中使用`nunjucks`类模板引擎语法实现`SEO`；`易上手`。

- 页面开发不依赖框架包装的任何模块，保持原生的`React`开发体验;`友好,易升级`。

- 数据获取由服务端统一处理加工，页面视图开发和数据加工分开处理；`逻辑更清晰`。

- 支持`SSR`和`CSR`动态调整，支持`SSR`缓存,降级。`高可用`。

- 支持其他`koa`开发框架使用。`可扩展`。

- 支持MPA,各页面组件可单独构建，`可页面级更新`。

