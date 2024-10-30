---
title: 鸿蒙应用路由Navigation介绍与项目应用
tags:
  - HarmonyOS
  - ArkTS
  - ArkUI
key: blog-comments
---
本文讲解鸿蒙应用的最新路由方案 [Navigation](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5) 的使用与在公司项目中的选型使用。

<!--more-->
## 前言
   鸿蒙官方在2024年05月11日发布了OpenHarmony SDK 5.0.0.22 (API 12 Canary3) , 在这次更新发布了最新的路由方案 [Navigation](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5) ,并建议开发者不再使用 router 页面导航，后续也将不再对 router 进行迭代维护。 具体可以查阅 [router和Navigation的技术选择、推荐使用场景和演进策略是什么](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs-V5/faqs-arkui-11-V5).
   
   下面我们从简单的例子学习 [Navigation](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5) 的使用。


## Navigation 主要概念

### Navigation 与 NavDestination

Navigation , 鸿蒙官方的中文名词是路由容器组件,其首选是一个组件,这个组件由导航页(NavBar)和子页(NavDestination)组成。导航页和子页的显示逻辑由页面显示模式来决定,分为单页面模式(NavigationMode.Stack)、分栏模式(NavigationMode.Split)、自适应模式(NavigationMode.Auto)。

![鸿蒙路由单页面模式示意图](/images/2024-05-19-鸿蒙路由单页面模式示意图.png)

单页面模式示意图

![鸿蒙路由分栏模式示意图](/images/2024-05-19-鸿蒙路由分栏模式示意图.png)

分栏模式示意图

由于公司的鸿蒙应用用户群体是手机用户,所以我们选择了单页面模式(NavigationMode.Stack),从示意图可以看到单页面模式下,页面由标题栏(Titlebar，包含菜单栏menu)、内容区(Navigation子组件或者NavDestination子组件)和工具栏(Toolbar)组成,当我们进行路由操作时候,就是使用新组件替换内容区。

### 路由关联与操作

有了上面的概念后,我们在这里做一个简单的路由使用举例

```
// 根容器入口页面
@Entry
@Component
struct App {
  @Provide('NavPathStack') entryHapARouter: NavPathStack = new NavPathStack()

  // 构建函数,每次进行路由操作会调用
  @Builder
  PagesMap(name: string) {
    if (name == 'FirstPage') {
      FirstPage()
    }
  }

  build() {
    Navigation(this.entryHapARouter) {
    }
    .mode(NavigationMode.Stack)
    .onAppear(() => {
      // 页面挂载后进行页面跳转
      this.entryHapARouter.pushPath(
        { 
          name: "FirstPage", 
          param: new Object({
                  paramA: 'A',
                  paramB: true
                })
        })
    })
    // 隐藏标题栏、菜单栏、工具栏
    .hideNavBar(true)
    .navDestination(this.PagesMap)
  }
}

// NavDestination 子页面组件
@Component
export struct FirstPage {
  build() {
     NavDestination() {
       Column() {
           Text("FirstPage NavDestination")
             .fontSize(20)
         }
         .justifyContent(FlexAlign.Center)
         .height('100%')
         .width('100%')
     }
     .backgroundColor('rgba(0,0,0,0.5)')
     .hideTitleBar(true)
   }
}

```

上面定义了一个入口页面,入口页面使用 Navigation 组件,给Navigation组件传递一个路由管理栈变量 entryHapARouter 和构建函数 PagesMap .并且定义了一个组件(子页面). 当使用路由栈变量进行路由操作时候,会触发构建函数 PagesMap,然后根据对应情况返回 NavDestination 组件替换内容区.

我们可以看到上面的路由关联模式有一个缺点,由于多模块项目里面子页面分布在不同模块,这个入口页面必须要依赖所有的子页面模块并且import,会导致不同模块之间的依赖耦合等问题。

所以官方提供了多模块项目的动态路由方案。动态路由方案提供系统路由表和自定义路由表两种方式。

- 系统路由表相对自定义路由表，使用更简单，只需要添加对应页面跳转配置项，即可实现页面跳转。

- 自定义路由表使用起来更复杂，但是可以根据应用业务进行定制处理。

### 动态路由之系统路由表

各业务模块（HSP/HAR）需要独立配置 router_map.json 文件，在触发路由跳转时，应用只需要通过NavPathStack提供的路由方法，传入需要路由的页面配置名称，此时系统会自动完成路由模块的动态加载、页面组件构建，并完成路由跳转，从而实现了开发层面的模块解耦。

```
@Builder
export function loginPageBuilder() {
  LoginPage()
}

@Component
export struct LoginPage {
  build() {
     NavDestination() {
       Column() {
           Text("LoginPage")
             .fontSize(20)
         }
         .justifyContent(FlexAlign.Center)
         .height('100%')
         .width('100%')
     }
     .backgroundColor('rgba(0,0,0,0.5)')
     .hideTitleBar(true)
   }
}
```

现在存在一个 `login`模块,并且在这个模块中存在一个登录页面 `LoginPage` .若我们需要跳转到这个模块的登录页面,那么需要声明其页面路由配置文件

在目标模块的配置文件 `module.json5` 添加路由表配置

```
{
  ...
  "module" : {
    "routerMap": "$profile:route_map"
  }
}
```

上面声明了模块下的 `resources/base/profile` 目录下的 router_map.json 文件是路由配置文件,内容如下

```
{
    "routerMap": [
      {
        "name": "LoginPage",
        "pageSourceFile": "src/main/ets/pages/LoginPage.ets",
        "buildFunction": "loginPageBuilder"
      }
    ]
}
```

我们可以看到声明一个路由目标页面,要声明3个变量,name,pageSourceFile(构造函数的文件路径),buildFunction(构造函数).

当进行路由操作的时候应用会从所有路由表中找到对应的name然后调用构建函数,替换内容区.

系统路由表使用注意事项

有时候构建会报错

```
Change the 'routerMap' object names listed below under routerMap in the respective router configuration files. Make sure the names are unique across the current module and the modules on which it depends.
```

这是因为应用存在重复依赖的情况,而重复依赖的模块存在路由声明,构建工具构建过程中会合并模块下的路由配置文件成为一个 `default-router-map.json` 文件,重复依赖会导致存在重复的 `routerMap` 对象.下面是构建打包后的hap解包之后的文件,能看到最终是合并成了一个 `default-router-map.json` 文件.

![鸿蒙hap解包路由](/images/2024-05-19-hap解包路由.png)


![鸿蒙合并路由json](/images/2024-05-19-鸿蒙合并路由json.png)

而官方对于这种情况 [关于多个har依赖问题](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs-V5/faqs-project-management-22-V5) 有过解释,当多个相同har/hsr重复依赖时候会出现会生成重复的har,这时候在最终合并 `default-router-map.json` 文件时候就会报错存在重复的 `routerMap` 对象.当项目发生上述报错的时候可以在终端输入 `ohpm list` 查看依赖关系,排除重复依赖的情况.

下面举例一个经典的重复依赖的情况示意图

![鸿蒙har重复依赖举例](/images/2024-05-19-鸿蒙har重复依赖举例.png)


如上面的重复依赖情形下,若模块C存在系统路由表,那么就会报错.我们需要把模块B对于模块C的依赖移除,或者将模块A对于模块C的依赖移除,两者择一.

### 动态路由之自定义路由表

上述的系统路由表,是构建工具帮我们合并管理路由表,并且会在需要进行跳转的页面自动 import 目标模块.这样会导致用户仅仅在页面没进行跳转操作时候,提前加载目标模块(初始化目标模块的静态变量、so库等),增加了应用的启动时间和内存.

而为了解决这个问题,官方提供自定义路由表,自定义路由表是开发者自己手动实现路由表,充当 `default-router-map.json` 文件的角色,并且让开发者利用 [动态 import](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-dynamic-import-V5) 的特性来达成按需延迟加载目标模块的目的.

下面是一个简单的示例如何实现自定义路由

```
export class RouterModule {
  // 手动实现路由表, WrappedBuilder支持@Builder描述的组件以参数的形式进行封装存储
  static builderMap: Map<string, WrappedBuilder<[object]>> = new Map<string, WrappedBuilder<[object]>>();

  // 初始化路由栈，需要关联Navigation组件
  static navPathStack: NavPathStack = new NavPathStack();

  // 注册页面组件到路由表，builderName是路由名字，builder参数是包裹了页面组件的WrappedBuilder对象
  public static registerBuilder(builderName: string, builder: WrappedBuilder<[object]>): void {
    RouterModule.builderMap.set(builderName, builder);
  }

  // 根据名字获取路由表中指定的页面组件
  public static getBuilder(builderName: string): WrappedBuilder<[object]> {
    const builder = RouterModule.builderMap.get(builderName);
    if (!builder) {
      console.info('not found builder ' + builderName);
    }
    return builder as WrappedBuilder<[object]>;
  }

  // 路由方法,通过获取页面栈跳转到指定页面
  // 通过传入RouterModule跳转到指定页面组件，RouterModule包含跳转需要的信息
  public static async push(builderName: string): Promise<void> {
    // 从builderName中获取包名harName
    const harName = builderName.split('_')[0];
    // 动态导包，包导入成功后调用包index页面的harInit方法，动态导入需要跳转的文件。该文件首次加载时将完成页面注册到RouterModule模块的builderMap中。
    await import(harName).then((ns: ESObject): Promise<void> => ns.harInit(builderName));
    // 通过路由名跳转，并携带路由页面所需信息param
    RouterModule.navPathStack.pushPath({ name: builderName });
  }
}
```

## 项目路由选型

官方提供了系统路由表方案和自定义路由表方案来实现跨模块路由,虽然自定义路由表方案实现了按需加载、延迟加载,减少主页面的启动时间和内存占用，提升性能.但动态导入需要开发者注意的事项略多,例如下面的使用注意事项,加重了开发者的心智负担。

- 需要在需要被跳转的模块入口声明开放函数给路由库调用，让需要进行跳转的模块调用基础路由库方法从而动态import被跳转的页面;

- 在主hap模块配置[动态import变量表达式加入编译](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-dynamic-import-V5#%E5%8A%A8%E6%80%81import%E5%8F%98%E9%87%8F%E8%A1%A8%E8%BE%BE%E5%BC%8F)

因此在现在手机性能普遍良好的情况下,我们团队选用了系统路由表方案。后续也会讲一下团队如何封装路由让开发者使用起来更方便的,这里的分享就到这,感谢大家.希望对大家有所启发.

## 参考文档

[鸿蒙 Navigation ](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5)

[鸿蒙官方最佳实践-应用导航设计](https://developer.huawei.com/consumer/cn/doc/best-practices-V5/bpta-application-navigation-design-V5)

[Navigation自定义动态路由示例](https://gitee.com/harmonyos-cases/cases/blob/master/CommonAppDevelopment/common/routermodule/README_AUTO_GENERATE.md)
