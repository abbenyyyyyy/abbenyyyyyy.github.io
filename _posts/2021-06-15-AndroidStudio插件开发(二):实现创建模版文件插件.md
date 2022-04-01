---
title: Android Studio 插件开发(二):实现创建模版文件插件
tags:
  - IDEA Plugin
  - Android Studio
key: blog-comments-2021-06-15
---
公司的 Android 项目架构是根据[谷歌Android应用架构指南](https://developer.android.google.cn/jetpack/guide)的 MVVM 架构变化而来,因此当新建一个 Activity 的时候需要同步新建 ViewModel 、 布局文件 ,甚至还有 Repository 文件, 在 Android Studio 4.0 之前,谷歌是提供了方法让我们输入相关参数快速新建这些模版文件的,但 Android Studio 4.0 之后之前的方法不再能使用,谷歌官方建议使用 IDEA 插件的形式来开发有快速创建模版文件功能的插件。因此有这次的学习 IDEA 插件开发的经历,之前的文章总结了 IDEA 插件开发的基础知识,本文在其基础上实现一个完整插件的开发,其功能为创建模版文件。
<!--more-->

## 梳理功能点与流程

### 功能点
- 自动读取用户右键选择的包名、文件夹名,自动赋值到新建配置弹窗;
- 根据用户在新建配置弹窗的输入自动创建 `xxxActivity.kt` 、 `xxxViewModel.kt` 、布局文件 `activity_xxx.xml` 一共3个文件;
- 将 新建的 Activity 注册到配置清单 `AndroidManifest.xml` 。

### 流程图

![Quick-Create-Android-Template流程图](/images/2021-06-15-Android流程图.png)

### 技术难点与解决

- 如何取得需要创建模版的文件夹对应的模块或项目的清单文件 `AndroidManifest.xml` 中的包名?

  从 [IntelliJ 的 Android 插件 Github 代码仓库](https://github.com/JetBrains/android) 其中的 [NewAndroidComponentDialog.java](https://github.com/JetBrains/android/blob/9a7e4762e4fe381575172311b9f4fb4d87280fea/android/src/org/jetbrains/android/actions/NewAndroidComponentDialog.java) 这个类中可以看到使用 `Manifest.getMainManifest(facet:AndroidFacet)` 可以获得 `AndroidManifest.xml` 的解析持有,然后 `getPackage` 可以获得清单文件 `AndroidManifest.xml` 中的包名,因此需要引入 `org.jetbrains.android` 插件。

- 如何取得需要创建模版的文件夹对应的路径包名?

  从 [IntelliJ 的 Android 插件 Github 代码仓库](https://github.com/JetBrains/android) 其中的 [NewAndroidComponentAction.kt](https://github.com/JetBrains/android/blob/9a7e4762e4fe381575172311b9f4fb4d87280fea/android/src/com/android/tools/idea/actions/NewAndroidComponentAction.kt) 这个类中可以看到使用 `fun AndroidFacet.getPackageForPath(moduleTemplates: List<NamedModuleTemplate>, targetDirectory: VirtualFile)` 可以获得文件夹的路径包名。

- 如何根据模版创建文件?

  [IDEA 插件开发官方指南之从模板创建文件](https://plugins.jetbrains.com/docs/intellij/using-file-templates.html#custom-create-file-from-template-actions)可知从模版创建文件有以下几步:
  1. 在 `resource` 目录中，创建 `fileTemplates.internal` 文件夹,用于存放模板文件，注意必须使用这个名称;
  2. 在 `fileTemplates.internal` 文件夹创建所需要的模板文件,带上拓展名,并且以 `.ft` 结尾,模版代码语法与 [Velocity](https://velocity.apache.org/) 一致;
  3. 在 `plugin.xml` 中公开声明模版文件,如你在 `fileTemplates.internal` 下的模版文件为 `QuickActivity.kt.ft` ,那么其声明如下
  ```xml
  <idea-plugin>
    ...
    <extensions defaultExtensionNs="com.intellij">
      ...
      <internalFileTemplate name="QuickActivity"/>
    </extensions>
    ...
  </idea-plugin>
  ```
  4. 在代码中使用 `FileTemplateUtil.createFromTemplate(template:FileTemplate,fileName:String,props:Properties,directory:PsiDirectory)` 来创建文件,值得注意的是,文件操作不要在主线程事件指派线程(Event Dispatcher Thread)上进行。

- 如何把 Activity 注册到清单文件 `AndroidManifest.xml` 中?

  [IntelliJ 的 Android 插件 Github 代码仓库](https://github.com/JetBrains/android) 其中的 [NewAndroidComponentAction.kt](https://github.com/JetBrains/android/blob/9a7e4762e4fe381575172311b9f4fb4d87280fea/android/src/com/android/tools/idea/actions/NewAndroidComponentAction.kt) 里的 `addToManifest` 函数里看出,可以使用 `Application..addActivity()` 来注册 Activity 。
  

### 实现效果

![演示视频](/images/2021-06-15-Quick-Create-Android-Template效果.mp4)

[文章项目代码仓库](https://github.com/abbenyyyyyy/Quick-Create-Android-Template),可供参考与编译,具体实现流程在下面的文章讲述。

## 创建 IDEA 插件项目

首先我们在 [IntelliJ Platform Plugin Template 仓库](https://github.com/JetBrains/intellij-platform-plugin-template) 点击 `Use this template` 可以将模版的一些配置初始化为仓库名保存到自己的 Github 仓库下。

修改根目录下的 `gradle.properties` 中的 `pluginSinceBuild` 、 `pluginUntilBuild` `platformVersion` 字段,以适应 Android Studio 的版本,查表 [Android Studio 对应 IDEA 版本](https://plugins.jetbrains.com/docs/intellij/android-studio-releases-list.html)可知 Android Studio 4.2 是基于 IntelliJ IDEA 2020.2.3,结合[IDEA 内部编号](https://plugins.jetbrains.com/docs/intellij/build-number-ranges.html)可知 IntelliJ IDEA 2020.2.3 为 202,因此我们必须要把插件支持的最低 idea 版本 `pluginSinceBuild` 赋值为 `202`,插件支持的 IDEA 平台 `platformVersion` 赋值为 `2020.2.3`.

修改根目录下的 `gradle.properties` 中的 `platformPlugins` 字段为 `org.jetbrains.kotlin, org.jetbrains.android`,增加官方 Kotlin 插件 、官方 Android 插件依赖,最后 `gradle.properties` 如下。

```properties
# IntelliJ 插件声明文件,参阅 https://plugins.jetbrains.com/docs/intellij/intellij-artifacts.html

# 插件组织
pluginGroup = com.github.abbenyyyyyy.quickcreateandroidtemplate
# 插件名字
pluginName = Quick-Create-Android-Template
# 插件版本
pluginVersion = 0.0.1

# idea版本信息请查阅 https://plugins.jetbrains.com/docs/intellij/build-number-ranges.html
# 插件支持的最低 idea 版本
pluginSinceBuild = 202
# 插件支持的最高 idea 版本
pluginUntilBuild = 213.*

# 支持的 IntelliJ IDEA平台: 社区版 , 参阅 https://github.com/JetBrains/gradle-intellij-plugin#intellij-platform-properties
platformType = IC
platformVersion = 2020.2.3

# 插件依赖 参阅 https://plugins.jetbrains.com/docs/intellij/plugin-dependencies.html
# 例子: platformPlugins = com.intellij.java, com.jetbrains.php:203.4449.22
platformPlugins = org.jetbrains.kotlin, org.jetbrains.android

# 用于编译源代码 Java 版本
javaVersion = 11
# 用于编译的 Gradle 版本
gradleVersion = 7.4

# 是否把 Kotlin 标准库打包进插件,默认请设置为 false . 因为用户的 idea 默认已有 kotlin.
# 参阅 https://plugins.jetbrains.com/docs/intellij/kotlin.html#kotlin-standard-library for details.
kotlin.stdlib.default.dependency = false
```

另外添加插件依赖还需要在 `plugin.xml` 中声明,告诉下载插件的人你的插件使用了哪些第三方插件,我们声明官方 Kotlin 插件 、官方 Android 插件,如下
```xml
<idea-plugin>
  ...
  <depends>com.intellij.modules.platform</depends>
  <depends>org.jetbrains.kotlin</depends>
  <depends>org.jetbrains.android</depends>
</idea-plugin>
```

修改 `build.gradle.kts` 添加 `runIde` 语法块,把调试插件时候启动的编辑器改为自己本地的 Android Studio 路径
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
到这里我们完成了插件项目的创建与初始配置。

## 添加新建模版文件Action

我们需要一个触发点来触发新建模版文件,我们选择将 Action 这个添加到用户右键的 `New` 菜单下,如何寻找 `New` 菜单的 `group-id` 呢,上一遍文章提到的,我们可以去[IDEA 社区版 Github](https://github.com/JetBrains/intellij-community),根据同级新建 Kotlin 文件的 Action 文本 `Kotlin Class/File` 作为搜索关键词,找到资源文件 [KotlinBundle.properties](https://github.com/JetBrains/intellij-community/blob/master/plugins/kotlin/frontend-independent/resources-en/messages/KotlinBundle.properties#L360) 里有这个文本,再以此根据使用该文本的代码方法 `KotlinBundle.message("action.new.file.text")` 搜索 [NewKotlinFileAction.kt](https://github.com/JetBrains/intellij-community/blob/master/plugins/kotlin/idea/src/org/jetbrains/kotlin/idea/actions/NewKotlinFileAction.kt#L47) 是新建 Kotlin 文件的 Action 。而最后使用 `org.jetbrains.kotlin.idea.actions.NewKotlinFileAction` 搜索,可以发现其在 [plugin.xml](https://github.com/yuchuangu85/intellij-community-idea/blob/f10d7ea81032c6b33a12d0728f5b4a5b0d742168/plugins/kotlin/resources-fir/META-INF/plugin.xml#L72) 中声明了该 Action 。结论:用户右键的 `New` 菜单的 `group-id` 为 `NewGroup` 。

最终我们新建 Action 文件 [NewAndroidTemplateAction.kt](https://github.com/abbenyyyyyy/Quick-Create-Android-Template/blob/main/src/main/kotlin/com/github/abbenyyyyyy/quickcreateandroidtemplate/actions/NewAndroidTemplateAction.kt),并在 `plugin.xml` 中声明如下
```xml
<idea-plugin>
  ...
  <actions>
      <!-- 行为动作(Action)静态注册 -->
      <action id="QuickCreateAndroidTemplate.NewAndroidTemplateAction"
          class="com.github.abbenyyyyyy.quickcreateandroidtemplate.actions.NewAndroidTemplateAction"
      >
        <!-- add-to-group 声明该行为动作按钮添加到右键新建菜单列表中 -->
        <add-to-group group-id="NewGroup"/>
      </action>
  </actions>
</idea-plugin>
```

## 显示配置弹窗

### 弹窗原型

![弹窗原型](/images/2021-06-15-Quick-Create-Android-Template原型.png)

### 代码实现

[NewActivityTemplateDialog.kt](https://github.com/abbenyyyyyy/Quick-Create-Android-Template/blob/main/src/main/kotlin/com/github/abbenyyyyyy/quickcreateandroidtemplate/dialogs/NewActivityTemplateDialog.kt) 继承 `DialogWrapper` 类,重写 `createCenterPanel()` 方法返回弹窗组件。

弹窗组件使用 Java Swing 的布局以及组件,。 在这里弹窗分解为上面无法编辑部分、下面可编辑部分(editContent 变量)。他们都使用了 `BoxLayout` 布局,值得注意的是所有在 `BoxLayout` 布局中的子组件都必须设置 `alignmentX` 为 `Component.LEFT_ALIGNMENT` ,否则默认居中显示。另外若出现布局错乱的情况,可以尝试给子组件设置最大高宽、默认高宽。

具体的可以可以参考 [Java Swing 布局与控件Oracle官方教程](https://docs.oracle.com/javase/tutorial/uiswing/layout/index.html) ,其中的例子与说明都很详尽。

## 创建文件与注册

在 [NewActivityTemplateDialog.kt](https://github.com/abbenyyyyyy/Quick-Create-Android-Template/blob/main/src/main/kotlin/com/github/abbenyyyyyy/quickcreateandroidtemplate/dialogs/NewActivityTemplateDialog.kt) 重写了 `doOKAction` ,实现用户点击确认后的操作,在进行了检查输入是否合法之后开始进行基于模版的文件创建以及注册。

前面的技术难点与解决已经讲解了如何使用代码创建模版文件,值得注意的是,文件操作不要在主线程事件指派线程(Event Dispatcher Thread)上进行,另外对于已有文件的修改(将Activity 注册到清单文件 `AndroidManifest.xml` 中),必须使用 `ApplicationManager.getApplication().runWriteAction(action:Runnable)` 来包裹实现,具体可参阅 [IDEA 插件开发官方指南之读写锁规则](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html)。

## 总结

文章讲解了如何实现快速创建 Activity 模版的插件如何开发,包括其中的难点。其实不难,难得是多方查询文档资料,多查多总结,才能成为一个合格、优秀的开发人员,希望你我共勉,感谢。

## 参考文档

[IntelliJ 的 Android 插件 Github 代码仓库](https://github.com/JetBrains/android)

[IDEA 插件开发官方指南之使用文件模版](https://plugins.jetbrains.com/docs/intellij/using-file-templates.html#improving-save-file-as-template-action)

[mvvm_clean_architecture_plugin 插件 Github 代码仓库 -- Senior](https://github.com/simplesoft-duongdt3/mvvm_clean_architecture_plugin)
