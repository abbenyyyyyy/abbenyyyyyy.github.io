---
title: 基于Puppeteer的自动化网页操作实践
tags:
  - 自动化
  - RPA
  - Node
  - nestjs
  - Puppeteer
---

公司的用户反馈处理业务使用了环信工单系统,主要业务流程是:

1. 客服在环信工单后台收到用户反馈,生成技术工单;
2. 技术工单同步到技术管理台;
3. 技术人员在技术管理台收到工单提醒,在技术管理台处理工单;
4. 然后技术人员还需把处理操作额外回填到环信工单后台(因环信通过 api 回填是额外增值服务,未采购)。

从上面的流程总结,技术人员处理一次工单,需要在 **技术管理台** 与 **环信工单后台** 进行两次重复操作,使用技术手段实现自动化操作,解放操作人员,为工作赋能是我们技术人员的基本能力,因此有这次的使用 Puppeteer 的自动化网页操作实践。

## 自动化常见技术原理

自动化操作基本原理可理解为三个操作的相互结合

1. 查找 控件、元素等操作主体;
2. 读取 位置、文本、状态等信息;
3. 控制 拖拉、点击、输入等操作。

基于实现基础原理可得出以下技术方案

1. 基于分解操作,使用爬虫技术侵入型操作；
2. 基于录制脚本,模拟用户在桌面环境的操作;
3. 基于浏览器 DevTool,模拟用户在浏览器的操作.

第一种技术方案有违法风险,所以不采用。

第二种技术方案,常见的是采用 微软在 Windows 操作系统中提供的核心应用程序编程接口([Win32 API](https://docs.microsoft.com/en-us/windows/win32/api/)) 来模拟用户在特定的仅供机器人使用的 Window 环境实现自动操作。重量级,需采购 Window 环境,开发语言基本为 c 或 c++ , 服务器成本与技术成本都高,基于我们的业务场景都是在浏览器场景下,所以不采用。

第三种技术方案,常见的是使用 [FireFox 浏览器的 DevTools API](https://developer.mozilla.org/en-US/docs/Tools/DevToolsAPI) 、 [谷歌 chrome 浏览器 DevTool API](https://developer.chrome.com/docs/devtools/) , 来模拟用户在浏览器的操作实现自动操作。采用该方案。

## 技术方案选型

确定了技术方案原理,采用基于浏览器 DevTool,模拟用户在浏览器的操作的 Github 开源方案有 [TagUI](https://github.com/kelaberetiv/TagUI) 、 [Puppeteer](https://github.com/puppeteer/puppeteer) 等,这里只展开谈我所了解的这两个库.

![对比总结.png](/images/2021-09-22-rpa方案对比.png)

## 基础知识与流程

- 基础知识

  [Puppeteer](https://github.com/puppeteer/puppeteer) 是基于 [谷歌 chrome 浏览器 DevTool API](https://developer.chrome.com/docs/devtools/) 的操作,因此原则上 [谷歌 chrome 浏览器 DevTool API](https://developer.chrome.com/docs/devtools/) 能实现实现的操作都能使用 [Puppeteer](https://github.com/puppeteer/puppeteer) 执行。

  常用命令

  1. [document.querySelector](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/querySelector) , 返回文档中与指定选择器或选择器组匹配的第一个 [HTMLElement](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement) 对象 , 常使用该命令确定网页上我们要操作的元素.

     ```javascript
     document.querySelector(".ui-itm-table span");
     ```

  2. [page.$(selector)](https://pptr.dev/#?product=Puppeteer&version=v10.4.0&show=api-pageselector) , [Puppeteer](https://github.com/puppeteer/puppeteer) 中获得操作元素的基本方法.

- 环信回填流程

  ![回填工单流程.jpeg](/images/2021-09-22-回填工单流程图.jpeg)

## 实际效果

此处为方便演示于调试,启动了有画面的 Chromium 浏览器 ,实际在生产环境采用无 GUI 的 Chromium 浏览器。

视频步骤分解

1. 启动后端服务;
2. 请求服务接口,传递工单 id、工单回复文本格式、工单回复文本;
3. 服务端从接口接收任务,启动 Chromium 浏览器 执行自动操作：登录环信客服工作台,打开对应工单,填充回复内容。

<iframe width="560" height="315" src="https://www.youtube.com/embed/M6OnBlYtFs4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## 总结

综上,我们利用 [Puppeteer](https://github.com/puppeteer/puppeteer) 来实现解放重复操作的结果,后期该技术也可用于网页重复性任务,如安卓应用发布到应用市场,需要登录各大应用市场管理后台执行发布操作等。希望此文能对各位有所启发，感谢。

## 代码参考

由于项目代码涉及公司业务,这里只展示一下具体的逻辑。

```javascript
import { Injectable, Logger } from '@nestjs/common';
import * as puppeteer from 'puppeteer';
import { FillEngagement } from './dto/fill-engagement.dto';

@Injectable()
export class OperationService {
  private readonly logger = new Logger(OperationService.name);

  async fillEngagement(fillEngagement: FillEngagement) {
    const startTime = new Date().getTime();
    this.logger.debug('开启执行任务 %d', startTime);
    const windowWidth = 1920;
    const windowHeight = 1440;
    const browser = await puppeteer.launch({
      headless: false,
      defaultViewport: { width: windowWidth, height: windowHeight },
      args: ['–no-sandbox', `--window-size=${windowWidth},${windowHeight}`],
    });
    const page = await browser.newPage();
    await page.setViewport({ width: windowWidth, height: windowHeight });
    await Promise.all([
      page.goto('https://099130.kefu.easemob.com/mo/signin'),
      page.waitForSelector('input[name=username]'),
    ]);
    // 填入账号、密码,执行登录
    await page.type('input[name=username]', process.env.HX_USER_NAME);
    await page.type('input[name=password]', process.env.HX_PW);
    await page.$eval('#em-auth-login-auto', (check) => {
      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      check.checked = true;
    });
    // 点击确定按钮进行登录
    const loginButtonElement = await page.$('.ui-cmp-btn');
    // 等待页面跳转完成
    await Promise.all([
      loginButtonElement.click(),
      page.waitForNavigation({
        // 500毫秒内不再有网络连接
        waitUntil: 'networkidle0',
      }),
    ]);
    // 跳转客服工作台
    await Promise.all([
      page.click('li[sign=agent]'),
      page.waitForNavigation({
        // 500毫秒内不再有网络连接
        waitUntil: 'networkidle0',
      }),
    ]);
    // 展开工单按钮
    await page.click('dl[name=mytickets]');
    // 点击显示'我的为解决工单列表',且等待工单列表加载完成
    await Promise.all([
      page.click('label[title=工单列表]'),
      page.waitForSelector('.ui-itm-table-history'),
    ]);
    // 开始寻找要填充的工单
    // 计算 工单编号 在父元素 li 的位置
    let childIndex = 0;
    const menuSpanList = await page.$$('.ui-itm-table span');
    for (let i = 0; i < menuSpanList.length; i++) {
      const title = await menuSpanList[i].getProperty('title');
      if (title.toString().indexOf('工单编号') != -1) {
        childIndex = i;
        break;
      }
    }
    const noElements = await page.$$(
      `.ui-itm-table-history :nth-child(${childIndex + 1})`,
    );
    // 工单回复正文的输入iframe的选择器
    const iframeSelector = 'iframe.cke_wysiwyg_frame';
    for (const noElementItem of noElements) {
      const noText = await noElementItem.getProperty('textContent');
      if (noText.toString().indexOf(fillEngagement.ticketId) != -1) {
        // 点击跳转要填充的工单页面
        await Promise.all([
          noElementItem.click(),
          page.waitForSelector(iframeSelector),
        ]);
        break;
      }
    }
    const iframeElement = await page.$(iframeSelector);
    const frame = await iframeElement.contentFrame();
    await frame.waitForSelector('body[contenteditable]>p');
    const innerContent =
      fillEngagement.formatType == 0
        ? `<p>${fillEngagement.content}</p><br/>`
        : fillEngagement.content;
    await frame.evaluate((innerContent) => {
      // html格式正文参考: <p>你好,该问题请参考&nbsp;<a data-cke-saved-href="https://www.baidu.com" href="https://www.baidu.com">操作手册</a>&nbsp;, 希望能对你有所帮助,谢谢。测试</p>
      // 插入工单回复正文
      document.querySelector('body[contenteditable]>p').innerHTML =
        innerContent;
    }, innerContent);
    // 点击提交回复,且等待提交请求完成
    await Promise.all([
      page.click('.emticket-comment-buttons>.ui-cmp-icontxtbtn'),
      // 刷新回复内容的接口,触发了说明提交回复成功
      page.waitForRequest((res) => res.url().indexOf('comments?page=') !== -1),
    ]);
    this.logger.debug(`完成提交表单任务 ${new Date().getTime() - startTime}`);
    // 关闭浏览器
    await browser.close();
  }

}
```

## 参考文档

[Puppeteer API 参考文档](https://pptr.dev/)  
[结合项目来谈谈 Puppeteer -- 张佃鹏](https://zhuanlan.zhihu.com/p/76237595)  
[Puppeteer 滑块登陆处理 -- papermoon](https://segmentfault.com/a/1190000021911714)
