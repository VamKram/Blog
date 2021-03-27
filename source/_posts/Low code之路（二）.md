
---
title: Low Code 之 自动填写
date: 2021/01/27
cover: https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=962309492,3418279211&fm=26&gp=0.jpg
categories:
- LowCode
tags: 
- puppeteer

---

# Low Code 之 自动填写

上次的blog部分完成了对Form的处理，这次记录一下，对数据获取方面设计的全过程，这次的工作也就是完成图中，对应每一个国家进行填写的这部分。

![image-20210327155304930](https://technologybook.tech/assets/img/image-20210327155304930.png)

我需要做的就是一个基于JSON 的可配置填写脚本们只需要通过简单的指令就能获取到页面信息并且操作页面进行处理。

## 调研

寻找能够控制页面访问的库，我把方向锁定在爬虫技术希望通过爬虫确定一个底层的库，然后基于底层的库进行封装。

**基于python的selenium**

>  本质上是一个测试框架，提供很好的的浏览器兼容性

**Puppeteer**

> Puppeteer is a Node library which provides a high-level API to control [headless](https://developers.google.com/web/updates/2017/04/headless-chrome) Chrome or Chromium over the [DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/). It can also be configured to use full (non-headless) Chrome or Chromium.

它能够

- 生成页面 PDF。
- 抓取 SPA（单页应用）并生成预渲染内容（即“SSR”（服务器端渲染））。
- 自动提交表单，进行 UI 测试，键盘输入等。
- 创建一个时时更新的自动化测试环境。 使用最新的 JavaScript 和浏览器功能直接在最新版本的Chrome中执行测试。
- 捕获网站的 [timeline trace](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/reference)，用来帮助分析性能问题。
- 测试浏览器扩展。



找到Pupeteer就基本上确定了，以Pupeteer为基础的自动填写脚本平台的设计。

一个通用的填写平台主要有以下几个困难

- 页面状态不稳定（页面可能有基于各种技术平台的，jquery到angular，以及目标构造表单方式）
- 合理有效的错误处理机制（包括应用程序错误和业务流程错误）
- 页面的校验机制
- 信息源错误过期等异常的上报
- 反反爬虫机制

对与 json schema 的配置我使用的是类似distributor的方式

![image-20210327175051537](https://technologybook.tech/assets/img/image-20210327175051537.png)

每一个动作对应一个处理函数

每个个type对应一种动作，这样配置文件就可以

```json
{
	type: "JUMP",
	target: "selector"
	option: {
		data: x
	}
}
```

一个distributor对一个的各个部分都处理完成之后

进入一个状态机器

大体分三步

对应的是[网页端目标的状态， 脚本状态， 服务端接口状态]

status

- SUCCESS target✅ script✅ server✅
- INTERNALERROR script❌
- FAILED target ❌
- SERVERERROR ❌

。。。

错误处理使用EventEmit对不同的错误CODE做处理

如图示

![image-20210327180759086](https://technologybook.tech/assets/img/image-20210327180759086.png)

错误处理机制如上图，

大体上分为

- retry
- 上报错误信息
- 慢启动重试
- 最后进入平台处理

## 页面校验部分

区分用户的有效性

有效性大概分为两部分

信息有效性

> 即当前信息是否滞后

数据有效性

> 记录国家的错误率，然后针对国家尽心处理

原则是 **由近到远** **由高到低**即有限处理距离当前日期近的身份信息和国家成功率高的

### 反反爬虫

- 平台验证码识别

接入平台验证码识别

- ip池防止ip封锁

### 插件机制

代码提供一种类似webpack的插件机制，在constructor中执行，分别对IP进行相应的处理，以达到快速切换IP线程池的操作。是一种中间件机制。

```typescript
class Distributor {
  constructor(private plugins){
    this.plugins = plugins
		....
  }
  process(){
    const ips = startSetIp({
      IP: this.plugins.ipTag
    })
    new this.plugins.Captcha()
    ...
  }
  startSetIp({IP}) {
    try {
      const ipIns = new IP();
      const newIP = ipIns.getNewestIpList()
      this.establish(newIP);
      ...
    } catch(){}
  }
}
    
    
    class Captcha {
      constructor(c) {
        this.c = c
        ...
      }
        cut(){...}
       async identify(image) {
          const result = await this.identify(image);
        }
    }
```

相关的代码还有很多，这里仅梳理一下结构，如果验证有效会考虑开源

***TODO***



