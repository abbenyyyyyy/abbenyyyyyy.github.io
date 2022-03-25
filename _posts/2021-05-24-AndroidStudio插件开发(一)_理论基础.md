---
title: IDEA 插件开发(一):理论基础
tags:
  - IDEA Plugin
  - Android Studio
key: blog-comments-2021-05-24
---

公司的 Android 项目架构是根据[谷歌Android应用架构指南](https://developer.android.google.cn/jetpack/guide)的 MVVM 架构变化而来,因此当新建一个 Activity 的时候需要同步新建 ViewModel 、 布局文件 ,甚至还有 Repository 文件, 在 Android Studio 4.0 之前,谷歌是提供了方法让我们输入相关参数快速新建这些模版文件的,但 Android Studio 4.0 之后之前的方法不再能使用,谷歌官方建议使用 IDEA 插件的形式来开发有快速创建模版文件功能的插件。因此有这次的学习 IDEA 插件开发的经历,本文总结了 IDEA 插件开发的基础知识,后续将在本文的基础上实现一个完整插件的开发。

<!--more-->

## 创建 IDEA 插件项目

Android Studio 是在 IDEA 的基础上增加了开发 Android 的插件以及功能改造而成的编辑器,因此开发 Android Studio 插件就是开发 IDEA 插件。而开发 IDEA 插件一般使用 IDEA 社区版或者旗舰版,因为其自带了 `IntelliJ Plugin Develop kit`,类似如 `Adnroid SDK` 的开发组件。

新建 IDEA 插件项目项目通用的有两种方式,一种是根据 IDEA 的引导创建一个新的 Gradle 项目,另外一种是使用 [IntelliJ Platform Plugin Template](https://github.com/JetBrains/intellij-platform-plugin-template) 模版库。根据 IDEA 的引导创建一个新的 Gradle 项目可以参考[官方图文讲解](https://plugins.jetbrains.com/docs/intellij/gradle-prerequisites.html#creating-a-gradle-based-intellij-platform-plugin-with-new-project-wizard)来一步一步实现。这里我们仅讲解 [IntelliJ Platform Plugin Template](https://github.com/JetBrains/intellij-platform-plugin-template) 模版库的形式,模版库可以充当一个 IDEA 插件项目脚手架,其中预设了一系列用于调试、编译打包、发布插件的 gradle 命令,以及一系列默认配置等。

首先我们在 [IntelliJ Platform Plugin Template 仓库](https://github.com/JetBrains/intellij-platform-plugin-template) 点击 `Use this template` 可以将模版的一些配置初始化为仓库名保存到自己的 Github 仓库下。

## 文件架构

根据模版初始化到自己仓库的 IDEA 插件项目文件结构如下

```
.
├── .github/                    GitHub Actions 工作流,含提交后自动发布等自动操作
├── .run/                       调试配置文件,一般不对其作修改
├── gradle
│   └── wrapper/                使用到的 Gradle 版本定义文件
├── build/                      生产构建输出目录,常用是编译打包后的 zip 文件在其中的 distributions 文件夹
├── src                     
│   └── main
│       ├── kotlin/             插件开发 kotlin 文件的文件夹
│       └── resources/      
│              ├── plugin.xml   定义了插件的 id、名字、使用到的第三方插件依赖、插件的'四大组件'声明的配置文件
│              └── ...  
│   └── ...
├── build.gradle.kts            定义插件的编译使用到的插件、插件版本适配平台、版本、使用到的第三方依赖、编译与调试 gradle 命令等编译配置文件
├── CHANGELOG.md                历史版本变更日志
├── gradle.properties           build.gradle.kts 使用到的常量声明
├── qodana.yml                  Qodana 代码静态检查配置文件,一般不使用代码静态检查功能,不对其作修改
└── ...
```

## "四大组件"

这里所谓的"四大组件"并不是官方说法,而是在开发插件过程中常用到的插件功能,而使用到这些功能都需要在 `plugin.xml` 文件中声明静态注册,类似 Android 开发中的四大组件,所以我这里使用"四大组件"来统称它们,只要了解掌握这"四大组件"就能开发实现90%的功能。下面来逐一讲解这些需要静态注册的组件。

### 服务(Services)

服务(Services)是指在这一整个完整的生命周期内不会重复实例化,能在对应的生命周期开启的时候自动实例化,并且在实例化的时候自动注入 Project 变量的类。一般用于封装一些在插件项目中使用到的可重用变量。

服务(Services)分为 应用程序级服务(applicationService)、项目级服务(projectService)以及一个不推荐使用的模块级服务(moduleService)。

应用程序级服务(applicationService),其跟随整个 IDEA 工作的生命周期,会在用户每次重新打开 IDEA 的时候实例化。

项目级服务(projectService),每当你使用 IDEA 打开一个新 Project 时候都会自动实例化一个对应该 Project 的服务实例。

值得注意的是不要在实现服务的时候应避免在其初始化过程中进行耗时过长的任务,否则会影响 IDEA 启动以及打开项目的速度。下面是服务(Services)在 `plugin.xml` 文件中静态注册的参考代码。

```xml
<idea-plugin>
    <id>org.jetbrains.plugins.template</id>
    <name>Template</name>
    <vendor>JetBrains</vendor>

    <depends>com.intellij.modules.platform</depends>

    <extensions defaultExtensionNs="com.intellij">
        <!-- 应用程序级服务(applicationService)静态注册 -->
        <!-- serviceImplementation的值为你项目包名+你项目中的实现服务类 -->
        <applicationService serviceImplementation="org.jetbrains.plugins.template.services.MyApplicationService"/>
         <!-- 项目级服务(projectService)静态注册 -->
        <projectService serviceImplementation="org.jetbrains.plugins.template.services.MyProjectService"/>
    </extensions>

    ...
</idea-plugin>
```

### 行为动作(Action)

行为动作(Action)是插件用户触发调用插件功能的通用方式,声明行为动作(Action)后会将相对应的按钮绘制在菜单项或工具栏,用户可以点击触发调用插件功能。

自定义的行为动作(Action)需要基础抽象类 `AnAction` ,并实现其中的 `actionPerformed(e: AnActionEvent)` 方法,以响应用户的行为操作。也可重写 `update(e: AnActionEvent)` 方法,根据具体情况是否显示该行为动作(Action)按钮。

同样的行为动作(Action)也需要在 `plugin.xml` 文件中静态注册,以下是参考代码。

值得注意的是其中的 `icon` IDEA 官方建议是尽可能地重用[官方内置在 IDEA 中的图标](https://jetbrains.design/intellij/resources/icons_list/),在[官方内置在 IDEA 中的图标列表](https://jetbrains.design/intellij/resources/icons_list/)点击心仪的图标后底部会出现对应的在 IDEA 中的文件路径,可以直接引用。

另外就是其中的 `add-to-group` 是声明该行为动作按钮添加到哪一个 IDEA 已有的菜单或工具栏中,可惜的是官方并没有提供已有的菜单或工具栏的 id,这里提供一个比较笨的方法,可以去[IDEA 社区版 Github](https://github.com/JetBrains/intellij-community),根据菜单文本或关键词 `action` 搜索,慢慢找到对应的菜单或工具栏 id ,如果有更好的方法欢迎分享,谢谢。

![idea-icons](/images/2021-05-24-idea-icons-list.png)

 ```xml
<idea-plugin>
    <id>org.jetbrains.plugins.template</id>
    <name>Template</name>
    <vendor>JetBrains</vendor>

    <depends>com.intellij.modules.platform</depends>

    <extensions defaultExtensionNs="com.intellij">
        <applicationService serviceImplementation="org.jetbrains.plugins.template.services.MyApplicationService"/>
        <projectService serviceImplementation="org.jetbrains.plugins.template.services.MyProjectService"/>
    </extensions>
    <actions>
        <!-- 行为动作(Action)静态注册 -->
        <action id="cn.abbenyyy.show_git_config_dialog" class="cn.abbenyyy.ConfigAction"
                text="设置" icon="AllIcons.Gutter.Colors">
            <!-- add-to-group 声明该行为动作按钮添加到已有的菜单或工具栏中,这里是添加到顶部的 Tools 菜单的第一个位置 -->
            <add-to-group group-id="ToolsMenu" anchor="first"/>
        </action>
    </actions>
</idea-plugin>
```

### 拓展(Extensions)

拓展(Extensions)是开发 IDEA 插件时候拓展 IDEA 原有功能的方式,不同 IDEA 平台的可供拓展的功能有 1000 多个,可以具体参考[官方说明](https://plugins.jetbrains.com/docs/intellij/extension-point-list.html),这里使用给 IDEA 添加工具窗口(与 IDEA 右侧 gradle 按钮显示的工具窗口同级)作为例子:

1. 新建 `WindowFactory.kt` 实现 `ToolWindowFactory` 接口;
2. 在 `plugin.xml` 文件中静态注册该自定义工具窗口
```xml
<idea-plugin>
    <id>org.jetbrains.plugins.template</id>
    <name>Template</name>
    <vendor>JetBrains</vendor>

    <depends>com.intellij.modules.platform</depends>

    <extensions defaultExtensionNs="com.intellij">
        <applicationService serviceImplementation="org.jetbrains.plugins.template.services.MyApplicationService"/>
        <projectService serviceImplementation="org.jetbrains.plugins.template.services.MyProjectService"/>
        <!-- 静态注册自定义工具窗口, -->
        <toolWindow factoryClass="org.jetbrains.plugins.template.toolWindow.WindowFactory"
                    id="Custom ToolWindow"
                    anchor="right"
                    icon="AllIcons.Nodes.Toolbox"
        />
    </extensions>
    ...
</idea-plugin>
```

### 监听者(Listeners)

监听者(Listeners)是 IDEA 插件开发过程中监听消息总线传递的消息的类,一般可监听 IDEA 的生命周期变化,也常用于监听项目文件的变化。在我开发 IDEA 插件的过程中使用的场景不多,因此就不多说了,可以参考[官方对于监听者(Listeners)](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html)的讲解。

## UI布局与控件

IDEA 的 UI 是基于 Java Swing 的,虽然官方也在推 [Kotlin UI DSL](https://plugins.jetbrains.com/docs/intellij/kotlin-ui-dsl-version-2.html) ,但是其 Kotlin UI DSL Version 1 很难用，只支持几个组件,没有良好的复用封装,而 Kotlin UI DSL Version 2 支持的 IDEA 平台版本过高(至少要2021.3),因此常用的还是 Java Swing。

如果要掌握绘制 IDEA 插件 UI 需要掌握的是 Java Swing 的布局以及组件使用方式,在这方面可以参考 [Java Swing 布局与控件Oracle官方教程](https://docs.oracle.com/javase/tutorial/uiswing/layout/index.html),其中有不少示例与说明,在这里就不赘述了。

## 线程规则

IDEA 线程可区分为主线程、后台线程等多个线程,其中主线程作为事件指派线程(Event Dispatcher Thread),也会简写为 EDT,该线程同时也负责 UI 绘制,不应该在 EDT 上进行耗时过长的任务阻塞线程,如版本控制操作、文件操作、网络操作等。

那么我们该如何跳转到后台线程执行操作呢。可以使用 `ApplicationManager.getApplication().executeOnPooledThread(action:Runnable)` 方法,在 `Runnable` 里的 `run()` 方法里进行后台线程操作。也可以使用 `ProgressManager.getInstance().run(task:Task)`, `task` 赋值为 `Task.Backgroundable` 的实例,这个是同步显示 IDEA 的底部进度条的后台耗时任务。

另外有时候我们还需要在在后台线程执行完耗时任务,需要跳转回 EDT 执行页面绘制或弹窗通知用户,这时候我们可以使用`GuiUtils.invokeLaterIfNeeded(runnable:Runnable,modalityState:ModalityState)` , 把 `modalityState` 赋值为 `ModalityState.defaultModalityState()` , 就会在 EDT 执行该 `runnable` 任务。

## 调试与编译发布

在开发 IDEA 插件的过程中,可以使用 `runIde` gradle 命令来调试插件,该命令会启动 IDEA 以及自动安装你开发中的插件。若想修改该命令,可以在项目根目录的 `build.gradle.kts` 中的修改 `runIde` 语法块.如下是设定 `runIde` 启动 Android Studio 的示例。

```kotlin
...
tasks {
 ...
 runIde {
        // 这里设定 runIde 的时候启动的编辑器路径
        // 下面是 mac OS 下的 Android Studio 路径,可以在 mac OS 的应用程序里查看
        ideDir.set(file("/Applications/Android Studio.app/Contents"))
        // 下面是 Window OS 的 Android Studio 路径示例,只需要指向 Android Studio 的安装根目录就行
        // ideDir.set(file("D:\\Android Studio"))
    }
}
```
执行 `runIde` gradle 命令方式:
   - 打开右侧 gradle 工具窗口,点击 `intelliJ -> runIde` 执行调试
   - 或者执行命令 `./gradlew runIde` 执行调试

编译插件可以使用 `buildPlugin` gradle 命令,编译完成后在 `./build/distributions` 目录下生成 zip 后缀的插件安装包。
执行 `buildPlugin` gradle 命令方式:
   - 打开右侧 gradle 工具窗口,点击 `intelliJ -> buildPlugin` 执行编译
   - 或者执行命令 `./gradlew buildPlugin` 执行编译

如果给公司内部使用可以把 zip 后缀的插件安装包分享然后从本地安装插件,但如想发布插件到插件市场可以需要先配置好给插件签名,具体可以参考[IDEA 官方指南](https://plugins.jetbrains.com/docs/intellij/plugin-signing.html),签名后才可以把安装包提交上传到 IDEA 官方插件市场,便携说明等,经过约2天等待就可以审核通过发布。

## 总结

该文汇总了开发 Android Studio 插件的基础知识,肯定有未详尽之处,可以参考官方的指南,后续会出如何实现一个完整的 IDEA 插件的文章,具体功能应该会是快速创建 MVVM 框架的模版文件,感谢阅读。

## 参考文档

[IDEA 插件开发官方指南](https://plugins.jetbrains.com/docs/intellij/welcome.html)

[IDEA 插件开发高级指南 -- Nazmul Idris](https://developerlife.com/2021/03/13/ij-idea-plugin-advanced/)

[Java Swing 布局与控件Oracle官方教程](https://docs.oracle.com/javase/tutorial/uiswing/layout/index.html)