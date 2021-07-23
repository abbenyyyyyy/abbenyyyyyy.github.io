---
title: Android静态代码扫描实践—1、静态代码扫描概述与初识ktlint
---

## 背景与问题

多人协作的项目中，基于代码稳定性以及代码安全（不安全的API使用和逻辑编写）的考虑，必然要出现团队内的代码风格规范，而为了解决“代码编写遵从风格规范全凭开发人员自觉、缺乏提示、检查和卡点机制”这一问题，静态代码扫描检查在团队协作的软件工程项目是必不可缺的一环。

## 问题举例

- 日志打印
  必须使用统一封装的打印方法，禁止使用System.out.print\android.util.Log，方便release版本禁止在 logcat 输出信息出现数据泄露的情况。

  错误示范
  ```
  class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ......
        Log.d("writelog", "start activity")
        // ......
    }

  }
  ```

- setHostnameVerifier 方法调用
  禁止调用 setHostnameVerifier 方法设置 ALLOW_ALL_HOSTNAME_VERIFIER 属性，以避免中间人攻击劫持，应用使用 STRICT_HOSTNAME_VERIFIER 属性。

  错误示范
  ```
  class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ......
        val schemeregistry = SchemeRegistry()
        val sslsocketfactory = SSLSocketFactory.getSocketFactory()
        // set STRICT_HOSTNAME_VERIFIER
        sslsocketfactory.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER)
        // ......
    }

  }
  ```

- 还有诸如必须给继承Activity、Fragment的文件添加注释方便理解；Activity必须继承Activity基础类；Fragment必须继承Fragment基础类等团队开发规范问题。

## 静态代码扫描工具的选择

由于我们团队开发已实现全Kotlin化，Kotlin常用的静态代码扫描方案有[Detekt](https://github.com/detekt/detekt)以及[ktlint](https://github.com/pinterest/ktlint)。

对比如下：
![对比](images/Android静态代码扫描实践1-img1.png)

由于我这边并不需要分析潜在性能与bug问题的功能，以及基于轻量化的目的，采用的是Kotlin官方推荐的[ktlint](https://github.com/pinterest/ktlint)工具。

## ktlint的集成与使用

我们采取了项目中使用Gradle集成ktlint的方式来集成ktlint规则，可参考[ktlint主页](https://github.com/pinterest/ktlint)。

### 在项目根目录下的app目录下的build.gradle添加如下配置
```
  ...
  configurations {
    ktlint
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
  ...
  task ktlint(type: JavaExec, group: "verification") {
    description = "Check Kotlin code style."
    classpath = configurations.ktlint
    main = "com.pinterest.ktlint.Main"
    args "-a", "src/**/*.kt", "--reporter=html,output=${buildDir}/ktlint.html"
  }
  check.dependsOn ktlint

  task ktlintFormat(type: JavaExec, group: "formatting") {
    description = "Fix Kotlin code style deviations."
    classpath = configurations.ktlint
    main = "com.pinterest.ktlint.Main"
    args "-F", "src/**/*.kt"
  }
```

### 使用方法运行gradle任务执行
- 静态检查代码是否符合规范
  
  Mac或者Lunix系统执行：./gradlew ktlint;   
  window系统执行：gradlew ktlint；  
  会执行代码检查任务，然后会在./app/build/文件夹生成ktlint.html报告。  
  ![html报告](images/Android静态代码扫描实践1-img2.png)

- 自动修改代码符合规范

  Mac或者Lunix系统执行：./gradlew ktlintFormat;   
  window系统执行：gradlew ktlintFormat；  
  会执行自动修改代码符合规范任务。  

## 总结

通过上文，我们理解了为何要进行静态代码扫描以及使用ktlint对项目代码进行扫描检查是否符合Kotlin官方代码风格规范，当然在实际实践中要如何限制团队成员遵守规范，毕竟不可能强行要求团队成员每次都使用gradle命令检查代码，这部分内容是我们下一步要讲解的内容。
如果我的文章对你有帮助或启发，辛苦大佬们点个赞👍🏻，支持我一下。
如果有错漏，欢迎大佬们指正，也欢迎大家一起讨论，感谢。