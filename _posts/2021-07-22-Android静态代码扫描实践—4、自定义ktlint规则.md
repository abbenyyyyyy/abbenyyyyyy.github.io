---
title: Android静态代码扫描实践—4、自定义ktlint规则
tags:
  - 代码规范
  - ktlint
  - Android
key: blog-comments-2021-07-22
---

前面3篇文章，我们介绍了静态代码扫描在团队的重要性以及在实际团队实践中如何使用Gitlab CI/CD配合静态代码扫描实现让团队成员低感知地遵守代码规范。而在之前我们的实践中仅仅是使用了 [ktlint](https://github.com/pinterest/ktlint) 实现了[Kotlind的官方代码风格规范](https://kotlinlang.org/docs/coding-conventions.html)检查，但在实际开发过程中，我们还会有更多团队中的代码规范，如日志打印方法的统一、每个activity文件必须要有注释等。  
因此，作为Android静态代码扫描实践的收官文章，我将带着大家如何使用 [ktlint](https://github.com/pinterest/ktlint) 写出自定义规则。

<!--more-->

## ktlint加载规则的流程
虽然官方也有简单的文档教我们如何自定义ktlint规则，但是我觉得先搞懂执行 ```./gradlew ktlint``` 后如何加载规则，更有助于我们自定义ktlint规则。

首先先找到定义 ```ktlint``` 这个gradle任务的地方,在项目根目录下的app目录下的build.gradle里面
```gradle
  ...
  configurations {
    ktlint
  }
  ...
  task ktlint(type: JavaExec, group: "verification") {
    description = "Check Kotlin code style."
    classpath = configurations.ktlint
    main = "com.pinterest.ktlint.Main"
    args "-a", "src/**/*.kt", "--reporter=html,output=${buildDir}/ktlint.html"
  }
  ...
  dependencies {
    ...
    ktlint("com.pinterest:ktlint:0.41.0") {
        attributes {
            attribute(Bundling.BUNDLING_ATTRIBUTE, getObjects().named(Bundling, Bundling.EXTERNAL))
        }
    }
    ...
  }
```
其中定义了一个 name 为 ktlint 的 gradle 任务，类型为 [JavaExec](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.JavaExec.html)，执行后将会在子进程中执行 Java 应用程序（Jar），classpath 定义了要执行的 Jar 的路径，而 ```configurations.ktlint``` 是一个定义好的名为 ktlint 的引用集合，在这里面仅引用了 ```"com.pinterest:ktlint:0.41.0"``` ，后续你可以添加自己的Jar。 ```main``` 表示要执行的 main 方法为 ```com.pinterest.ktlint.Main```.  

因此我们可以直接在[ktlint](https://github.com/pinterest/ktlint)的源码看到  [com.pinterest.ktlint.Main方法](https://github.com/pinterest/ktlint/blob/a0a146262bdc5fedd772e044e4bc8052a5c1f924/ktlint/src/main/kotlin/com/pinterest/ktlint/Main.kt#L54)

![main方法.png](/images/2021-07-22-main方法.png)

我们执行 ```./gradlew ktlint``` 的时候并没有带子命令，因此直接进入下一步 ```ktlintCommand.run()``` 方法。

![ktlintCommand-run方法.png](/images/2021-07-22-ktlintCommand-run方法.png)

其中 ```failOnOldRulesetProviderUsage()``` 是判断使用的 Jar 是否有继承老的规则方法，如果有，直接报错。而后续就是我们要找的加载规则的方法。
```kotlin
val ruleSetProviders = rulesets.loadRulesets(experimental, debug, disabledRules)
```
再进入看看[loadRulesets方法](https://github.com/pinterest/ktlint/blob/a0a146262bdc5fedd772e044e4bc8052a5c1f924/ktlint/src/main/kotlin/com/pinterest/ktlint/internal/RuleSetsLoader.kt#L12)

![load-RuleSetProvider方法.png](/images/2021-07-22-load-RuleSetProvider.png)

可以看到加载了 Jar 里所有实现 ```RuleSetProvider``` 抽象类的类，当然还有一些过滤条件，而 ```RuleSetProvider``` 抽象类 ```get``` 方法返回了一系列实现 ```com.pinterest.ktlint.core.Rule``` 抽象类的规则类， 对后面的步骤还感兴趣的，大家可以去看 [ktlint](https://github.com/pinterest/ktlint) 的源码，这里我们只需要了解加载规则的流程。粗略总结如下图：

![ktlint加载规则流程](/images/2021-07-22-ktlint加载规则流程.png)

因此我们自定义规则就是自定义一个类实现 ```RuleSetProvider``` 抽象类，在这个类中返回自定义的规则集合，然后导出成 Jar , 然后在项目根目录下的app目录下的build.gradle里面通过 ```ktlint``` 引用你的 Jar。

## 程序结构接口 (PSI)

上面简单介绍了 ktlin 如何加载自定义规则，了解后明白我们需要自定义一个类实现 ```RuleSetProvider``` 抽象类，在这个类中返回自定义的规则集合，而规则是一个实现 ```com.pinterest.ktlint.core.Rule``` 抽象类，在这个实现了规则的类中的 ```visit``` 抽象方法，在这个方法里面我们要完成识别不符合规范的代码块并输出警告提醒文本的功能，而该抽象方法的 ```ASTNode``` 参数就是我们识别代码块的关键。  

```ASTNode``` 是 JetBrains 对于旗下 IDE 的抽象语法树（Abstract Syntax Tree，AST）的实现 -- PSI（程序结构接口）其中的一个类。以树状的形式表现编程语言，将我们程序员所编写的源代码语法结构进行抽象表示。可以理解为 PSI 将程序员编写的代码转换为方便进行代码语法分析的树状结构代码。

而我们可以使用 [PsiViewer插件](https://plugins.jetbrains.com/plugin/227-psiviewer) 来直观的查看通过 PSI 生成的树状结构，下面两张图可以直观的看出该插件的使用以及树状结构的展示:

![PSI示例1](/images/2021-07-22-PSI示例1.png)

![PSI示例2](/images/2021-07-22-PSI示例2.png)

## 实现你的第一个自定义 ktlint 规则

「Talk is cheap. Show me the code」. 因此这里我用一个自定义的 ktlint 规则 -- 不可直接继承 Activity()  , 必须继承 BaseActivity 的实现当示例，希望大家能从中了解如何实现自定义规则，[示例代码Github](https://github.com/abbenyyyyyy/ktruleset).为方便调试，下面示例是在一个可用的 Android 项目下进行，这样方便我们调试，完成开发后，可以迁移到一个独立的 kotlin 项目，方便分发使用，如[示例代码Github](https://github.com/abbenyyyyyy/ktruleset).

### 创建自定义 ktlint 规则模块

- 在项目根文件夹中与app模块处于同一文件夹级别创建一个单独的模块，这里我将模块命名为 custom_rules；

![新建rules模块](/images/2021-07-22-新建rules模块.png)

- 将新建模块下的 ```build.gradle``` 文件修改如下，其中依赖的 kotlin 版本号要与项目根目录的 ```build.gradle``` 文件的版本一致：
```gradle
plugins {
    id 'kotlin'
}

compileKotlin {
    kotlinOptions.jvmTarget = "1.8"
}
compileTestKotlin {
    kotlinOptions.jvmTarget = "1.8"
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.5.20"
    compileOnly "com.pinterest.ktlint:ktlint-core:0.41.0"
}
```

- 告诉 ktlint 查找到我们实现了 ```RuleSetProvider``` 的类.在新建模块下的 ```src->main``` 下面新建文件夹 ```resources/META-INF/services```，并且在该目录下新建 ```com.pinterest.ktlint.core.RuleSetProvider``` 文件，在文件中添加
```
com.tc.custom_rules.CustomRuleSetProvider
```

这时候我们的文件目录如图：

![自定义ktlint规则step1menu](/images/2021-07-22-自定义ktlint规则step1menu.png)

### 新建规则类实现规则

- 新建 ```ExtendBaseRule``` 类实现 ```Rule``` 抽象类，其中 id 是方便我们查找过滤该规则的， ASTNode 的相关可以参考 [IDEA 程序结构接口 (PSI) 官方参考文档](https://plugins.jetbrains.com/docs/intellij/psi.html) ，结合 [PsiViewer插件](https://plugins.jetbrains.com/plugin/227-psiviewer) 我们可以清楚如何筛选不符合规则的 kotlin 文件
```kotlin
package com.tc.custom_rules

import com.pinterest.ktlint.core.Rule
import com.pinterest.ktlint.core.ast.ElementType
import com.pinterest.ktlint.core.ast.children
import org.jetbrains.kotlin.com.intellij.lang.ASTNode
import org.jetbrains.kotlin.psi.stubs.elements.KtStubElementTypes

class ExtendBaseRule : Rule("kclass-extend-base-rules") {
    override fun visit(
        node: ASTNode,
        autoCorrect: Boolean,
        emit: (offset: Int, errorMessage: String, canBeAutoCorrected: Boolean) -> Unit
    ) {
        if (node.elementType == KtStubElementTypes.CLASS) {
            println("使用调试打印日志：${node.text}")
            //修饰符为 class 的ASTNode
            var isExtendActivity = false
            //判断该class是否继承了Activity
            for (childNode in node.children()) {
                if (childNode.elementType == KtStubElementTypes.SUPER_TYPE_LIST) {
                    //psi中继承与实现的类
                    for (minChild in childNode.children()) {
                        if (minChild.elementType == KtStubElementTypes.SUPER_TYPE_CALL_ENTRY) {
                            //psi中继承的类，判断继承的ASTNode的文本
                            if (minChild.text == "Activity()") {
                                isExtendActivity = true
                            }
                            break
                        }
                    }
                }
            }
            if (isExtendActivity) {
                //该class是继承了Activity，再判断是不是BaseActivity
                for (childNode in node.children()) {
                    if (childNode.elementType == ElementType.IDENTIFIER) {
                        //第一个标识符，是类名
                        if (isExtendActivity && childNode.text != "BaseActivity") {
                            //该class是继承了Activity，也不是BaseActivity，因此输出错误
                            emit(
                                childNode.startOffset,
                                "Activity请继承BaseActivity！",
                                false
                            )
                            break
                        }
                        break
                    }
                }
            }
        }
    }
}
```

- ```CustomRuleSetProvider``` 类实现 ```RuleSetProvider``` ，返回上面定义的规则
```kotlin
package com.tc.custom_rules

import com.pinterest.ktlint.core.RuleSet
import com.pinterest.ktlint.core.RuleSetProvider

class CustomRuleSetProvider : RuleSetProvider {
    override fun get(): RuleSet = RuleSet(
        "custom-rule-set",
        ExtendBaseRule()
    )
}
```

### 与 ktlint 共同使用

- 在 ```app``` 模块的 ```build.gradle``` 依赖模块
```
...
dependencies {
  ...
  ktlint project(':custom_rules')
}

```

- 然后终端执行 ```./gradlew ktlint``` ,可以看到我们自定义的规则已经产生作用
![执行ktlint日志](/images/2021-07-22-执行ktlint日志.png)

![ktlint报告](/images/2021-07-22-ktlint报告.png)

### 导出自定义规则的 jar

实际实践中，我们并不可能每次有新项目配置规则的时候都添加一个自定义规则模块，因此我们需要把自定义规则模块导出成 jar ，方便 Android 项目引用。

你可以在刚才的自定义规则模块基础上执行
```
./gradlew :custom_rules:build
```
或者把刚才的自定义模块独立成一个 kotlin 项目，执行
```
./gradlew build
```
可以在 ```build->lib``` 中看到构建出的 jar , 之后就可以发布到 Maven 仓库了。

## 总结

讲解了如何实现自定义规则，基于 ktlint 和 Gitlab CI/CD 的团队静态代码规范实践这个系列基本上也完结了。

## 参考文档
[Gradle 参考文档](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.JavaExec.html)  
[Writing your first ktlint rule -- Niklas Baudy](https://medium.com/@vanniktech/writing-your-first-ktlint-rule-5a1707f4ca5b)  
[IDEA 程序结构接口 (PSI) 官方参考文档](https://plugins.jetbrains.com/docs/intellij/psi.html)  
[自定义规则示例代码Github](https://github.com/abbenyyyyyy/ktruleset)