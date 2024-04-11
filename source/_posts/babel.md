title: Babel自用笔记
date: 2020/03/05
cover: https://technologybook.tech/assets/img/ast1.png
categories:
- compiler
tags:
- babel

---

# babel自用笔记

### 关于抽象语法树的可以看这个[之前写blog](https://technologybook.tech/2020/01/02/ast/)。

## Babel 的处理步骤

Babel **解析（parse）**，**转换（transform）**，**生成（generate）**三个步骤 。

![](https://technologybook.tech/assets/img/ast3.png)

### Visitors

>  关于[访问者模式](https://refactoringguru.cn/design-patterns/visitor/typescript/example).的介绍。
>
> **访问者**是一种行为设计模式， 允许你在不修改已有代码的情况下向已有类层次结构中增加新的行为。



这是一个简单的例子，是数字111转变为string 123。

```javascript
module.exports = ({types: t}) => {
  return {
    visitor: {
      NumericLiteral (path) {
        fs.writeFileSync('./t.json', JSON.stringify(path.node, 2))
        if (path.node.value === 111) {
          path.replaceWith(t.stringLiteral('123'))
        }
      },
    },
  }
}

```

这是一个访问者，在遍历中时，如果遇到一个 NumericLiteral 即数字的时候会调用 NumericLiteral()` 方法。

这些调用都发生在**进入**节点时，不过有时候我们也可以在**退出**时调用访问者方法。.

假设我们有一个树状结构：

![image-20210321210154324](https://technologybook.tech/assets/img/ast.png)

上面的AST是一个简单的sayHi函数产出的结果。

```javascript
function sayHi() {
	var hi = "hi!";
	console.log(hi)
}
```

### Paths

从上可以看出AST 的结构节点都异常复杂，节点之间的互相关联关系可以通过Path-一个简便的快速查找节点的方式来进行查找。.

**Path** 是表示两个节点之间连接的对象。

例如，这是path.node包含的信息︰

```json
{
  "type": "NumericLiteral",
  "start": 11,
  "end": 14,
  "loc": {
    "start": {
      "line": 2,
      "column": 10
    },
    "end": {
      "line": 2,
      "column": 13
    }
  },
  "extra": {
    "rawValue": 111,
    "raw": "111"
  },
  "value": 111
}
```

如果整体的话：

```json
{
  "type": "VariableDeclarator",
  "start": 7,
  "end": 14,
  "loc": {
    "start": {
      "line": 2,
      "column": 6
    },
    "end": {
      "line": 2,
      "column": 13
    }
  },
  "id": {
    "type": "Identifier",
    "start": 7,
    "end": 8,
    "loc": {
      "start": {
        "line": 2,
        "column": 6
      },
      "end": {
        "line": 2,
        "column": 7
      },
      "identifierName": "a"
    },
    "name": "a"
  },
  "init": {
    "type": "NumericLiteral",
    "start": 11,
    "end": 14,
    "loc": {
      "start": {
        "line": 2,
        "column": 10
      },
      "end": {
        "line": 2,
        "column": 13
      }
    },
    "extra": {
      "rawValue": 111,
      "raw": "111"
    },
    "value": 111
  }
}
```



当然路径对象还包含增删改查节点的方法。



### Scopes

这里指 JavaScript 支持[词法作用域](https://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scoping_vs._dynamic_scoping)，通过tree的嵌套关系来确定作用域。

一个例子:

```javascript
function outFun2() {
    var inVariable = "inner";
}
outFun2();
console.log(inVariable); // Uncaught ReferenceError: inVariable is not defined

```

在babel插件编写的过程中对作用域的使用如下。

作用域会被用到的字段：

```
{
  path: path,
  block: path.node,
  bindings: [...]
  ...
}
```

当你创建一个新的作用域时，需要给出它的路径和父作用域，之后在遍历过程中它会在该作用域内收集所有的引用(“绑定”)。

一旦引用收集完毕，你就可以在作用域（Scopes）上使用各种方法，稍后我们会了解这些方法。

##  以下是编写插件的一些笔记

| 功能                    | BinaryExpression                          | field                                             |
| :---------------------- | ----------------------------------------- | ------------------------------------------------- |
| Salad                   | BinaryExpression                          | left/right (path.node.left = t.identifier("hi");) |
| 对当前对象的赋值        | path[x]                                   | path[x] = n                                       |
| 对字段的check           | t.is.x(path.node.y, { attr: z })          | y is x ? y.attr=== z ? True : false               |
| 查找最接近的父函数      | path.getFunctionParent                    | path.getStatementParent 一直向上                  |
| 替换节点                | path.replaceWith/path.replaceWithMultiple | path.replaceWith常用                              |
| 直接替换代码            | replaceWithSourceString                   | 一把梭                                            |
| 插入前后                | insertBefore/insertAfter                  | 插入                                              |
| 插入程序content中       | unshiftContainer/pushContainer            | 较常用                                            |
| 删除节点                | remove                                    | remove                                            |
| 本地变量是否绑定        | path.scope.hasBinding/hasOwnBinding       | 常用来检查scope body                              |
| 推送VariableDeclaration | push({ id： string, init: path.node });   | 不常用                                            |
| rename                  | path.scope.rename                         | 常用                                              |
| 抛错                    | throw path.buildCodeFrameError            |                                                   |
| 遍历path                | path.traverse                             | 尽量少用用 path[x] or path.get替代                |
|                         |                                           |                                                   |
|                         |                                           |                                                   |






