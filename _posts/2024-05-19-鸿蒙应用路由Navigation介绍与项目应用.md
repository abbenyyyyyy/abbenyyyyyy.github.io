---
title: 鸿蒙应用路由Navigation介绍与项目应用
tags:
  - HarmonyOS
  - ArkTS
  - ArkUI
key: blog-comments
---
本文讲解鸿蒙应用的最新路由方案 [Navigation](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5) 的使用与在公司项目中的使用。

<!--more-->
## 前言
   鸿蒙官方在2024年05月11日发布了OpenHarmony SDK 5.0.0.22 (API 12 Canary3) , 在这次更新发布了最新的路由方案 [Navigation](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5) ,并建议开发者不再使用 router 页面导航，后续也将不再对 router 进行迭代维护。 具体可以查阅 [router和Navigation的技术选择、推荐使用场景和演进策略是什么](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs-V5/faqs-arkui-11-V5).
   
   下面我们从简单的例子学习 [Navigation](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5) 的使用。


## Navigation 主要概念

### Navigation 与 NavDestination

Navigation , 鸿蒙官方的中文名词是路由容器组件,其首选是一个组件,这个组件由导航页(NavBar)和子页(NavDestination)组成。导航页和子页的显示逻辑由页面显示模式来决定,分为单页面模式(NavigationMode.Stack)、分栏模式(NavigationMode.Split)、自适应模式(NavigationMode.Auto)。

单页面模式示意图

![鸿蒙路由单页面模式示意图](/images/2024-05-19-鸿蒙路由单页面模式示意图.png)

分栏模式示意图

![鸿蒙路由分栏模式示意图](/images/2024-05-19-鸿蒙路由分栏模式示意图.png)

由于公司的鸿蒙应用用户群体是手机用户,所以我们选择了单页面模式(NavigationMode.Stack),从示意图可以看到单页面模式下,页面由标题栏(Titlebar，包含菜单栏menu)、内容区(Navigation子组件或者NavDestination子组件)和工具栏(Toolbar)组成,当我们进行路由操作时候,就是使用新组件替换内容区。

### 路由关联与操作

有了上面的概念后,我们在这里做一个简单的路由举例

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

我们可以看到上面的路由关联模式有一个缺点,由于多模块项目里面子页面分布在不同模块,这个入口页面必须要依赖所有的子页面模块并且import,会导致不同模块之间的依赖耦合，以及首页加载时间长等问题。

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

在目标模块的配置文件module.json5添加路由表配置

```
{
  ...
  "module" : {
    "routerMap": "$profile:route_map"
  }
}
```

上面声明了 模块下的 `resources/base/profile` 目录下的 router_map.json 文件是路由配置文件,内容如下

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

这是因为一个hap(应用)下多个相同har/hsr重复依赖时候会出现,需要检查不要出现重复依赖的情况.

### 动态路由之自定义路由表

自定义路由表方案是手动构建一个