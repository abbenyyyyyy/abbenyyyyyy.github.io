---
title: Android静态代码扫描实践—3、配置GitLab Runner与编写.gitlab-ci.yml
tags:
  - 代码规范
  - ktlint
  - Android
---

在前面的文章讲解了实际实践中如何配合 GitLab CI/CD 落实团队代码风格规范的统一，而其中 GitLab CI/CD 流程需要有 GitLab Runner 工具配套来跑对应项目要执行的流水线job。  
当然，配置 GitLab Runner 这件事你可以花10块钱请运维小哥哥喝杯蜜雪冰城，然后让他帮忙。如果你还想对于配置 GitLab Runner 与编写 .gitlab-ci.yml 有所了解，请继续阅读。

<!--more-->

## GitLab Runner简介
回顾一下 GitLab CI/CD 的工作流程：
- 首先，定义 .gitlab-ci.yml 文件。在这个文件中就定义了要执行的 job 和命令；
- 接着，将本地文件文件推送至远程仓库；
- 最后，远程仓库通知 Runner，执行 .gitlab-ci.yml 文件定义好的 job。

GitLab 与 Runner 之间通过 API 进行通信，因此只需要 Runner 所在的机器有网络并且可以访问 GitLab 服务器即可，也即是 Runner 可以不与 GitLab 在同一个环境内，一个 Runner 可以在另一个虚拟机、物理机、Docker 容器，或者一个容器集群内部署。

Runner 内部定义了许多可用于在不同场景中运行构建的执行器 Executors，常用的有Shell、Docker。

![Runner执行器对比](/images/2021-07-12-Runner执行器对比.png)

使用 Docker 执行器可以使用网络上丰富的 image 镜像，轻松创建具有依赖服务的构建环境，但下载 image 镜像在复杂的国内网络下会遇到较多问题。另外每次构建的时候都是一个新的环境，无法长久保存构建产生的文件，而我们团队每次构建的时候都会生成代码报告，并不想每次都把代码报告上传到另外的地方保存，因此我们团队选择了 Shell 执行器。

## GitLab Runner安装实践

1. 安装  
可在服务器上通过此命令查询服务器系统
```
lsb_release -a
```

此处实践服务器系统为 CentOS Linux release 7.6.181(core-4.1-amd64)。

添加GitLab的官方仓库
```
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
```
安装最新版本的 GitLab Runner
```
sudo yum install gitlab-runner
```

2. 注册,关联Gitlab
```
[root@localhost ~]#  gitlab-runner register
Running in system-mode.                            
# 引导会让你输入gitlab的url，输入自己的url，例如http://gitlab.example.com/                           
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://xxx.xxx.xxx:xxx/
# 引导会让你输入token，去相应的项目下找到token，例如xrjc3tWcdQpLcEwoYzkU
Please enter the gitlab-ci token for this runner:
xrjc3tWcdQpLcEwoYzkU
# 输入描述
Please enter the gitlab-ci description for this runner:
[localhost.localdomain]: develop
# 引导会让你输入tag，一个项目可能有多个runner，是根据tag来区别runner的，输入若干个就好了，比如web,hook,deploy,develop.需要记住，.gitlab-ci.yml 中使用到tag来区别调用哪个Runner.
Please enter the gitlab-ci tags for this runner (comma separated):
develop
# 是否运行未标记的版本
Whether to run untagged builds [true/false]:
[false]: false
# 是否将运行程序锁定到当前项目
Whether to lock Runner to current project [true/false]:
[false]: true
Registering runner... succeeded                     runner=xrjc3tWc
#  引导会让你输入executor类型，我们输入shell
Please enter the executor: shell, ssh, docker+machine, docker, docker-ssh, parallels, virtualbox, docker-ssh+machine, kubernetes:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
```

3. 配置完成  
配置完成后可在 Gitlab -> 设置  -> CI/CD -> Runner 展开中看见该 Runner。
![Runner配置完成](/images/2021-07-12-Runner配置完成.png)

## .gitlab-ci.yml

### .gitlab-ci.yml简介
通过配置项目根目录下的 .gitlab-ci.yml 文件来告诉 CI 要对你的项目做什么。一个  其中可以有多个 job ,这些 job 组合成一个完整的 pipeline 。
```yaml
# 此处定义完整的 pipeline 里的job，以及它们的执行顺序.
stages:
  - build
  - test

job-0:
  stage: build
  # tags定义了哪个Runner执行该 job.
  tags:
    - dsl
  # 该job具体执行命令.
  script:
    - echo "Check the java version, then build some Ruby project files:"
    # shell执行器下所有命令必须要在 Runnner 机器内能正常执行，也即是执行 java -version ，Runnner 机器必须配置好Java环境.
    - java -version

job-1:
  stage: test
  tags:
    - dsl
  script:
    - echo "If the files are built successfully, test some files with one command:"
    - ./gradlew app:assembleRelease
  # rules 规定该 job 在什么时候执行，此处定义为前置阶段 job-0 失败时候才会执行 job-1.
  rules:
    - when: on_failure
``` 
关于 .gitlab-ci.yml 文件的语法以及其自定义的变量这里不详细展开，大家想了解的可以查阅参考文档中的[.gitlab-ci.yml 文件官方文档](https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.html)。

### 简单的实现 ktlint 扫描任务的.gitlab-ci.yml
```yaml
# 定义后续使用到的变量，其中 GIT_STRATEGY 是告诉 Runner 如何在每次CI/CD（后称：作业）获取最新代码，设置为clone，每次作业将会都克隆一遍仓库
variables:
 GIT_STRATEGY: clone
 APP_NAME: "Test"
 WEIXIN_WEBHOOK: "企业微信机器人地址"
 MANAGER_PHONE: "135******4"
 REPORE_URL: "报告网址"
# 每次作业前执行的命令，这里是提权 gradlew 文件，给予执行权限
before_script:
 - chmod +x ./gradlew
# 定义流水线阶段，这里分4阶段 lint(静态检查)、notify_dev(通知合并开发者)、notify_manager(通知管理员)、notify_merge(公布合并)
stages:
 - lint
 - notify_dev
 - notify_manager
 - notify_merge
staticAnalysis:
 stage: lint
 tags:
   - dsl
 script:
   - ./gradlew ktlint
 rules:
   # 属于合并请求才执行该 job
   - if: $CI_PIPELINE_SOURCE == "merge_request_event"
notifyDev:
 stage: notify_dev
 tags:
   - dsl
 script:
   #运行脚本，通知提交人不符合规范，请根据报告文档修改后重新push
   - bash /usr/local/dslapp-reports/notifyDev.sh "$APP_NAME" "$WEIXIN_WEBHOOK" "$REPORE_URL" "$GITLAB_USER_NAME"
 rules:
   # 属于合并请求并且前一个执行的 job 失败时候才执行该 job
   - if: $CI_PIPELINE_SOURCE == "merge_request_event"
     when: on_failure
notifyManager:
 stage: notify_manager
 tags:
   - dsl
 script:
   - curl "${WEIXIN_WEBHOOK}" -H 'Content-Type:application/json' -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"**$APP_NAME有新的合并请求:**\n提交人:$GITLAB_USER_NAME\n合并请求标题：$CI_MERGE_REQUEST_TITLE\n请注意审核 \",\"mentioned_mobile_list\":[\"$MANAGER_PHONE\"]}}"
 rules:
   # 属于合并请求并且前一个执行的 job 成功时候才执行该 job
   - if: $CI_PIPELINE_SOURCE == "merge_request_event"
     when: on_success
notifyMerge:
 stage: notify_merge
 tags:
   - dsl
 script:
   - curl "${WEIXIN_WEBHOOK}" -H 'Content-Type:application/json' -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"**$APP_NAME的合并请求已批准:**\n被批准的合并请求标题：$CI_COMMIT_DESCRIPTION\"}}"
   - echo "合并请求被批准后:"+$CI_PIPELINE_SOURCE
 rules:
   #提交的描述有‘See merge request’关键词，说明是合并请求被批准后触发
   - if: '$CI_COMMIT_DESCRIPTION =~ /See merge request.*/ && ($CI_COMMIT_BRANCH == "master" || $CI_COMMIT_BRANCH == "dev")'
```
##  GitLab Runner 机器配置 Android 环境
GitLab Runner 机器必须要有能执行你定义的命令的环境，如果执行器为 Docker 的可以下载适合的 image。但我们选择的是 Shell 执行器.环境与安装 Runner 的机器并不隔离，使用的是同一个环境。因此我们需要提前配置 Android 环境，前面提过我这边安装 Runner 的机器系统是 CentOS Linux release 7.6.181(core-4.1-amd64)，下面配置能执行 ktlint 检查命令的环境。

- 配置jdk
  ```
  # 通过浏览器工具获取登陆后的jdk1.8的下载链接，再使用 wget 命令下载jdk1.8压缩包.
  wget https://download.oracle.com/otn/java/jdk/8u291-b10/d7fc238d0cbf4b0dac67be84580cfb4b/jdk-8u291-linux-x64.tar.gz?AuthParam=1626056251_3a0fae7d2e6b819092bf9ffb8d15db2d
  ```
  ```
  # 下载成功后解压至安装目录.
  tar -zxvf jdk-8u291-linux-x64.tar.gz -C /usr/local/
  ```
  ```
  # 编辑 /etc/profile 增加export JAVA_HOME=/usr/local/jdk1.8.0_291，export PATH=$PATH:$JAVA_HOME/bin，完成配置JAVA_HOME.
  vim /etc/profile
  ```
  ```
  # 重新加载系统环境变量.
  source /etc/profile
  ```
- 配置 androidSdk
  ```
  # 新建文件夹.
  mkdir /usr/local/android-sdk-linux
  ```
  ```
  # 下载[androidSdk仅命令行工具](https://developer.android.com/studio#command-tools).
  wget https://dl.google.com/android/repository/commandlinetools-linux-6858069_latest.zip
  ```
  ```
  # 解压下载的androidSdk仅命令行工具.
  unzip commandlinetools-linux-6858069_latest.zip
  ```
  ```
  # 编辑/etc/profile,新增export ANDROID_SDK_ROOT=/usr/local/android-sdk-linux，export PATH=$PATH:$ANDROID_SDK_ROOT/cmdline-tools.
  vim /etc/profile
  ```
  ```
  # 重新加载系统环境变量.
  source /etc/profile
  ```

## 总结
上文简单介绍了 Runner 和 .gitlab-ci.yml ，以及如何配置 Runner 以及 配置 Android 环境。后续将讲解如何自定义 ktlint 规则。 

## 参考文档
[GitLab Runner官方文档](https://docs.gitlab.com/runner/)  
[GitLab Runner安装官方文档](https://docs.gitlab.com/runner/install/linux-repository.html)  
[.gitlab-ci.yml 文件官方文档](https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.html)  
