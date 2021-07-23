---
title: Android静态代码扫描实践—2、ktlint与GitLab CI/CD的集成
tags:
  - 代码规范
  - ktlint
  - Android
---

前文讲解了为何要进行静态代码扫描以及使用ktlint对项目代码进行扫描检查是否符合[Kotlind的官方代码风格规范](https://kotlinlang.org/docs/coding-conventions.html)，在实践中要如何限制团队成员必须遵守规范，毕竟不可能强行要求团队成员每次都使用gradle命令检查代码，我们团队使用的是与GitLab CI/CD流程相结合。

## 代码检查阶段的选择

开发人员编写代码时候通常并不希望其过程频繁地被打断，因此在什么阶段进行代码检查以及如何快速修改符合规范，会影响开发人员开发效能以及对于规范的接受程度。  
一般开发人员进行项目开发涉及2个主要阶段，编辑器编写代码、代码托管。不同阶段检查工具与特点对比如下：

![提示阶段](/images/2021-07-08-提示阶段.png)
 
因此我们在实践中两个阶段都进行代码检查与提醒。
- 编码阶段，使用Android Studio 市场上面的的 [Ktlint](https://plugins.jetbrains.com/plugin/15057-ktlint-unofficial-)插件，安装后若代码不符合规范[Kotlind的官方代码风格规范](https://kotlinlang.org/docs/coding-conventions.html)会自动提示，以及可以一键格式化修改。

- 代码托管阶段，结合分支管理与GitLab CI/CD，当开发人员提出合并代码请求时候，CI/CD自动检查代码规范，若不符合规范提醒并且不合并该申请。

## GitLab CI/CD介绍

GitLab CI/CD 顾名思义是代码存储在 GitLab 的 Git 仓库中，GitLab 自带的一套Continuous Integration（持续集成）、Continuous Delivery（持续交付）流程方法。对于每次远程代码仓库的变动（推送、合并等），你都可以创建一组脚本来自动构建和测试你的应用程序。通过这套CI/CD流程，我们可以确保代码所引入的更改能通过你为应用程序建立的所有测试，准则和代码合规性标准。

GitLab CI/CD的工作流程：
![工作流程](/images/2021-07-08-工作流程.png)

由上图可以看到GitLab CI/CD 流程需要一个执行CI/CD脚本的工具以及一个定义CI/CD流程的根目录中的 .gitlab-ci.yml 文件。

- GitLab Runner，执行CI/CD脚本的工具，是与GitLab分离的，因此需要在GitLab管理后台中配置GitLab Runner。
- .gitlab-ci.yml 文件，指定构建、测试和部署的脚本，其根本上是一个 [YAML文件](https://en.wikipedia.org/wiki/YAML) 。


## 代码托管阶段实践流程

除了IDE内安装插件提醒，我们团队还在开发人员将代码合并到‘预发布分支’时候通过GitLab CI/CD自动检查，当然如果要求实效性比较高，也可以在每一次push代码到Gitlab的时候检查。

![合并代码流程](/images/2021-07-08-合并代码流程.png)


实现该流程的前置条件： 
  - Gitlab 将项目‘预发布分支’设置为受保护，避免开发人员直接push代码到‘预发布分支’，必须提出合并请求才能将‘功能分支’代码合并到‘预发布分支’；
  - Gitlab 在GitLab项目‘设置’ > ‘通用’并展开‘合并请求’中的‘合并检查’设置为流水线必须成功，这样当Gitlab CI/CD流水线不成功时候无法同意合并请求；
  - Gitlab配置了可用的GitLab Runner，可在GitLab项目‘设置’ > ‘CI/CD’并展开‘Runners’查看可用的Runner；
  - 配置可用的.gitlab-ci.yml 文件。

## 最终效果

看了上图，估计你应该对于整个流程有大概的了解，那么现在展示一段代码的在Gitlab走过的路。

step1：当开发人员在‘功能分支’开发完毕后，提出合并请求合并功能分支的代码到‘预发布分支’，触发流水线。
![step1](/images/2021-07-08-step1.png)

step2：代码不符合规范，流水线失败，触发企业微信通知，开发人员根据检查报告修改代码后重新提交代码到‘功能分支’可以再次触发流水线。
![step2失败通知](/images/2021-07-08-step2失败通知.png)

step3：代码符合规范，流水线成功，触发企业微信通知管理者进行code review后合并代码。
![step3检查成功通知](/images/2021-07-08-step3检查成功通知.png)

step4：合并代码成功后进行企业微信通知。
![step4合并通知](/images/2021-07-08-step4合并通知.png)

## 供参考.gitlab-ci.yml文件
 ```
 variables:
  GIT_STRATEGY: clone
  APP_NAME: "Test"
  WEIXIN_WEBHOOK: "https://baidu.com"
  MANAGER_PHONE: "135******4"
  REPORE_URL: "http://test.com/ktlint.html"
before_script:
  - chmod +x ./gradlew
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
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
notifyDev:
  stage: notify_dev
  tags:
    - dsl
  script:
    #运行脚本，通知提交人不符合规范，请根据报告文档修改后重新push
    - bash /usr/local/dslapp-reports/notifyDev.sh "$APP_NAME" "$WEIXIN_WEBHOOK" "$REPORE_URL" "$GITLAB_USER_NAME"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: on_failure
notifyManager:
  stage: notify_manager
  tags:
    - dsl
  script:
    - curl "${WEIXIN_WEBHOOK}" -H 'Content-Type:application/json' -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"**$APP_NAME有新的合并请求:**\n提交人:$GITLAB_USER_NAME\n合并请求标题：$CI_MERGE_REQUEST_TITLE\n请注意审核 \",\"mentioned_mobile_list\":[\"$MANAGER_PHONE\"]}}"
  rules:
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
## 参考文档
[Android静态代码扫描效率优化与实践](https://tech.meituan.com/2019/11/07/android-static-code-canning.html)  
[Gitlab 合并请求官方文档](https://docs.gitlab.com/ee/user/project/merge_requests/)  
[Gitlab CI/CD官方文档](https://docs.gitlab.com/ee/ci/)
