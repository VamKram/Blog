title: Express源码解析并实现
date: 2019/06/16
cover: https://technologybook.tech/assets/img/ex2.png
categories:
- source
tags:
- express

---

# Express源码解析

## 从http开始

关于http的文章可以看之前写的  -> [这个](https://technologybook.tech/2019/05/15/httpNhttps/)

## 到node http 模块

一个最简单的http server是这样的

```typescript
const http = require('http');
http
  .createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World!');
})
  .listen(8080, '127.0.0.1', () => {
  console.log(`server start`);
});
```



1. 创建 Server instance Server extends from net

![image-20210321012622791](https://technologybook.tech/assets/img/ex.png)

2. 监听request事件

![image-20210321012622791](https://technologybook.tech/assets/img/ex1.png)

3. parse TCP 数据流
4. http parse （解析 头部/body/是否解析完毕）
5. 保存 IncomingMessage 和 ServerResponse 实例 通过assignSocket 发送request事件参数为 req， res



## Express

### 一个express的例子

```javascript
const express = require("express");
const app = express();
app.get("/", (req, res) => {
  res.end("Hello World!");
})
app.listen(8080);
```

### express 源码架构

![image-20210321012622791](https://technologybook.tech/assets/img/ex2.png)

Express 对路由的设计遵循开放封闭，单一职责的原则，由Router负责处理与路由相关的一切问题，在Layer中分层负责path与行为的描述，

Route中进行处理

# 我的MiniExpress实现

- [x] middleware
- [x] Route.use
- [x] App route

等基础功能。

```typescript
import { NextHandler, Request, Response } from "../typings/project";
import Layer from "./Layer";
import Route from "./Route";
import { isMatch } from "../utils/checker";

export default class Router {
  public stack: Layer[] = [];
  middleware: { path: string, handler: NextHandler }[] = [];


  handle(req: Request, res: Response, handler: NextHandler) {
    let index = 0;
    let { pathname } = req;
    let next = () => {
      while (this.middleware.length) {
        const { path, handler }
        = this.middleware.shift() as { path: string, handler: NextHandler };
        if (isMatch(path, pathname)) {
          handler(req, res, next);
        }
      }
      if (index >= this.stack.length) return handler(req, res, () => {
      });
      let layer = this.stack[index++];
      if (layer.match(pathname)) {
        layer.handleRequest(req, res, next);
      } else {
        next();
      }
    };
    next();
  }

  route(path: string) {
    // 创建处理函数
    const route = new Route();
    // 匹配path 分配处理函数
    const layer = new Layer(path, route.dispatch);
    layer.route = route;
    this.stack.push(layer);
    return route;
  }

  get(path: string, handler: NextHandler = () => {
  }): void {
    const currentRoute = this.route(path);
    currentRoute.get(handler);
  }

  post(path: string, handler: NextHandler = () => {
  }): void {
    const currentRoute = this.route(path);
    currentRoute.post(handler);
  }

  use(path: string, handler: NextHandler = () => {
  }): void {
    this.middleware.push({path, handler});
  }
}
```



```typescript
import { Method, Request, Response } from "../typings/project";
import Route from "./Route";
import { isMatch } from "../utils/checker";

export default class Layer {
  route: Route = new Route;
  method: Method = Method.UNKNOWN;

  constructor(public path: string, public handler: Function) {
    this.path = path;
    this.handler = handler;
  }

  match(path: string) {
    return isMatch(this.path, path);
  }

  handleRequest(req: Request, res: Response, next: () => void) {
    this.handler(req, res, next);
  }
}
```



```typescript
import { Method, NextHandler, Request, Response } from "../typings/project";
import Layer from "./Layer";

export default class Route {
  stack: Layer[] = [];

  dispatch = (req: Request, res: Response, outNext: Function) => {
    let index = 0;
    const next = () => {
      if (index >= this.stack.length) return outNext();
      let layer = this.stack[index++];
      if (layer.method === req.method.toUpperCase()) {
        layer.handleRequest(req, res, next);
      } else {
        next();
      }
    };
    next();
  };

  get(handler: NextHandler) {
    let layer = new Layer("/", handler);
    layer.method = Method.GET;
    this.stack.push(layer);
  }

  post(handler: NextHandler) {
    let layer = new Layer("/", handler);
    layer.method = Method.POST;
    this.stack.push(layer);
  }
}
```

```typescript
import { AppBase, NextHandler, Request, Response } from "./typings/project";
import * as http from "http";
import { Server } from "net";
import { requestFormatter } from "./formatter/requestFormatter";
import { responseFormatter } from "./formatter/responseFormatter";
import Router from "./Router";
import { isFunc } from "./utils/checker";

abstract class App implements AppBase {
  router: Router = new Router();

  constructor(private protocol: any) {
    this.protocol = protocol;
  }

  abstract get(path: string, handler: NextHandler): void;
  abstract get(handler: NextHandler): void;

  abstract post(path: string, handler: NextHandler): void;
  abstract post(handler: NextHandler): void;

  abstract use(handler: NextHandler): void;
  abstract use(path: string, handler: NextHandler): void;

  listen(...params: Parameters<Server["listen"]>) {
    const server = (this.protocol as typeof http).createServer((streamReq, streamRes) => {
      const req: Request = requestFormatter(streamReq);
      const res = responseFormatter(streamRes);
      this.router.handle(req, res, (req: Request, res: Response) => {
        res.end(`Cannot find ${req.url}`);
      });
    });
    server.listen(...params);
  }
}

export default class Application extends App {
  get(path: string, handler: NextHandler): void;
  get(handler: NextHandler): void;
  get(path: string | NextHandler, handler: NextHandler = () => {
  }): void {
    if (isFunc(path)) {
      handler = path;
      path = "*";
    }
    this.router.get(path, handler);
  }

  post(path: string, handler: NextHandler): void;
  post(handler: NextHandler): void;
  post(path: string | NextHandler, handler: NextHandler = () => {
  }): void {
    if (isFunc(path)) {
      handler = path;
      path = "*";
    }
    this.router.post(path, handler);
  }

  use(handler: NextHandler): void;
  use(path: string, handler: NextHandler): void;
  use(path: NextHandler | string, handler: NextHandler = () => {
  }): void {
    if (isFunc(path)) {
      handler = path;
      path = "*";
    }
    this.router.use(path, handler);
  }
}

const app = new Application(http);
app.get("/", (req: Request, res: Response, next: any) => {
  console.log("I'm in 111", 111);
  next();
});

app.get("/", (req: Request, res: Response, next) => {
  console.log("I'm in 202", 2);
  res.end("Hello World");
});

app.listen(3121, () => {
  console.log(3121);
});

```

