title: webpack king（三）
date: 2019/09/17
cover: https://raw.githubusercontent.com/webpack-contrib/awesome-webpack/master/media/awesome_webpack_branding.png
categories:
- tool
tags:
- webpack

---
# webpack king（三）

webpack的设计与实现

以下是一个Mini webpack的执行流程图。

![w3](https://technologybook.tech/assets/img/w3.png)



通过acorn对源代码的编译得到Ast，以下是Ast节点所包含的信息，对于commonjs规范来说，只需要将callee下的name重写，并将argument重写就可以实现模块的有效引用

![w1](https://technologybook.tech/assets/img//w1.png)

![w2](https://technologybook.tech/assets/img/w2.png)

## 我的实现

```typescript
import * as path from 'path';
import * as fs from 'fs';
import * as process from 'process';
import * as babylon from "babylon";
import traverse from "@babel/traverse";
import generator from "@babel/generator";
import * as t from '@babel/types';
import { SyncHook } from 'tapable';
interface Config {
    entry: string,
    mode: "development" | "production",
    output: {
        filename: string,
        path: string
    },
    module?: {
        rules: Array<{test: RegExp, use: Array<string>}>
    },
    plugins?: Array<any>
}
type Modules = {[k in string]: string}

class Minipack {
    root: string = process.cwd();
    entry: string = this.config.entry;
    modules: Modules = {};
    hooks = {
        beforeStart: new SyncHook(["beforeStart"]),
        compile: new SyncHook(["compile"]),
        emit: new SyncHook(["emit"])
    }
    constructor(public config: Config) {
        this.config = config;
        let plugins = this.config.plugins || [];
        plugins.forEach(p => p.apply(this))
    }
    static CURRENT: Readonly<string> = "./";

    getSourceCode(path: string): string {
        let code = fs.readFileSync(path, 'utf8');
        code = this.load(path, code);
        return code;
    }

    parseCode(code: string, parentPath: string): {code: string, deps: Array<string>}{
        const ast = babylon.parse(code)as any as t.Node;
        let deps: Array<string> = [];
        traverse(ast , {
            CallExpression(p) {
                let node = p.node as any;
                if (node.callee .name === 'require') {
                    node.callee.name = "__webpack_require__";
                    const currentName = node.arguments[0].value;
                    let name = `${Minipack.CURRENT}${path.join(parentPath, currentName)}`;
                    deps.push(name)
                    node.arguments = [t.stringLiteral(currentName)];
                }
            }
        })
        const finalCode = generator(ast);
        return { code: finalCode.code, deps}
    }

    build(filePath: string, isEntry = false ): Modules {
        const code = this.getSourceCode(path.resolve(this.root, filePath));
        let moduleName = Minipack.CURRENT + path.relative(this.root, filePath);
        isEntry && (this.entry = moduleName);
        this.hooks.compile.call("compile")
        const {code: sourceCode, deps = []} = this.parseCode(code, path.dirname(filePath));
        this.modules[moduleName] = sourceCode;
        deps.forEach(dep => {
            this.build(dep);
        })
        return this.modules;
    }

    start() {

        const module = this.build(this.config.entry, true);
        console.log(module)
    }

    load(p: string, code: string) {
        let rules = this.config.module?.rules;
        if (!rules?.length) return code;
        rules.forEach(rule => {
            const {test, use } = rule;
            if (test.test(p)) {
                let loaderLen = 0;
                while(loaderLen <= use.length) {
                    let loader = require(use[loaderLen++]);
                    code = loader(code);
                }

            }
        })
        return code
    }
}

const m = new Minipack({
    mode: "development",
    entry: "./test.js",
    output: {
        filename: "bundle.js",
        path: path.resolve(__dirname, "dist")
    }
});
m.hooks.beforeStart.call("beforeStart")
m.start();
```

### 关于loader

一个简单的loader-它的作用是可以通过babel将代码转换。

```typescript
import * as loaderUtils from 'loader-utils';
import * as babel from 'babel-core';
function loader(code) {
    const opt = loaderUtils.getOptions(this);
    const fn= this.async()
    babel.transform(source, {
        ...opt
    }, (err, result) => {
        const {code, map} = result
        fn(err, code, map)
    })
}
```

在使用过程中还需注意 loaderContext的使用,即this上挂的一些API

**同步返回** 

this.callback

**异步返回**

```
const callback = this.async();
callback(err, code, map)
```

**配置了options对象**

this.query

**模块所在的目录**

this.context  

**解析出来的 request 字符串**

this.request

### “Raw” loader

资源文件会被转化为 UTF-8 字符串，然后传给 loader。

**`pitch` 方法**

如果某个 loader 在 `pitch` 方法中给出一个结果，那么这个过程会回过身来，并跳过剩下的 loader。

## Plugin

一个最简单的plugin

- compiler ：webpack 实例，记载着你在 webpack.config.js 中的配置和其它基础构建Module 等。
- compilation ：包含了当前的模块资源、编译生成资源、变化的文件等，继承compiler。

```typescript
class Plugin {
    apply(packer: Minipack) {
        packer.hooks.emit.tap("emit", () => {
            console.log('emit');
        })
    }
}
```

插件生命周期

entryOption : 在 webpack 选项中的 entry 配置项 处理过之后，执行插件。
afterPlugins : 设置完初始插件之后，执行插件。
compilation : 编译创建之后，生成文件之前，执行插件。。
emit : 生成资源到 output 目录之前。
done : 编译完成。`compiler.hooks` 下指定**事件钩子函数**，便会触发钩子时，执行回调函数。

Webpack 提供三种触发钩子的方法：

- `tap` ：以**同步方式**触发钩子；
- `tapAsync` ：以**异步方式**触发钩子；
- `tapPromise` ：以**异步方式**触发钩子，返回 Promise；
- ![image-20210324010907998](https://technologybook.tech/assets/img/w4.png)