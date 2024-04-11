title: A abstract S syntax T tree
date: 2020/01/02
cover: https://technologybook.tech/assets/img/ast.png
categories:
- compiler
tags:
- ast

---

# AST 抽象语法树



## 什么是AST？

> 是[源代码](https://zh.wikipedia.org/wiki/源代码)[语法](https://zh.wikipedia.org/wiki/语法学)结构的一种抽象表示。它以[树状](https://zh.wikipedia.org/wiki/树_(图论))的形式表现[编程语言](https://zh.wikipedia.org/wiki/编程语言)的语法结构，树上的每个节点都表示源代码中的一种结构。之所以说语法是“抽象”的，是因为这里的语法并不会表示出真实语法中出现的每个细节。比如，嵌套括号被隐含在树的结构中，并没有以节点的形式呈现；而类似于 `if-condition-then` 这样的条件跳转语句，可以使用带有三个分支的节点来表示。 
> -- From wiki

AST是代码片段具体语义的抽象表达可以与目标代码之间相互转换的一种中间状态。

## Javascript Compiler

我们最常用的Javascript compiler babel首页是这样描述的。

> Babel is a toolchain that is mainly used to convert ECMAScript 2015+ code into a backwards compatible version of JavaScript in current and older browsers or environments. Here are the main things Babel can do for you:
>
> - Transform syntax
> - Polyfill features that are missing in your target environment (through [@babel/polyfill](https://babeljs.io/docs/en/babel-polyfill))
> - Source code transformations (codemods)
> - And more! (check out these [videos](https://babeljs.io/videos.html) for inspiration)

babel 的 parser traverse是基于Ecmascript标准实现的，很重要的一点就是其中节点类型。

以下是 [ECMAScript](https://262.ecma-international.org/7.0/#sec-grammar-summary)支持的词法类型.

![image-20210321210154324](https://technologybook.tech/assets/img/ast1.png)



## AST的数据结构

了解一个parser的结构，以及parser解析语法所依赖的规则后，接下来，我们需要了解一下一个parser所生产出来的结果——抽象语法树。在文章的开头，我有简单的解释抽象语法树即是具体代码片段的抽象表达，那它具体是长什么样的呢？

```javascript
function sayHi() {
	var hi = "hi!";
	console.log(hi)
}
```

以上的代码片段，AST树结构如下：

```json
{
  "type": "Program",
  "start": 0,
  "end": 54,
  "body": [
    {
      "type": "FunctionDeclaration",
      "start": 0,
      "end": 54,
      "id": {
        "type": "Identifier",
        "start": 9,
        "end": 14,
        "name": "sayHi"
      },
      "expression": false,
      "generator": false,
      "async": false,
      "params": [],
      "body": {
        "type": "BlockStatement",
        "start": 17,
        "end": 54,
        "body": [
          {
            "type": "VariableDeclaration",
            "start": 20,
            "end": 35,
            "declarations": [
              {
                "type": "VariableDeclarator",
                "start": 24,
                "end": 34,
                "id": {
                  "type": "Identifier",
                  "start": 24,
                  "end": 26,
                  "name": "hi"
                },
                "init": {
                  "type": "Literal",
                  "start": 29,
                  "end": 34,
                  "value": "hi!",
                  "raw": "\"hi!\""
                }
              }
            ],
            "kind": "var"
          },
          {
            "type": "ExpressionStatement",
            "start": 37,
            "end": 52,
            "expression": {
              "type": "CallExpression",
              "start": 37,
              "end": 52,
              "callee": {
                "type": "MemberExpression",
                "start": 37,
                "end": 48,
                "object": {
                  "type": "Identifier",
                  "start": 37,
                  "end": 44,
                  "name": "console"
                },
                "property": {
                  "type": "Identifier",
                  "start": 45,
                  "end": 48,
                  "name": "log"
                },
                "computed": false,
                "optional": false
              },
              "arguments": [
                {
                  "type": "Identifier",
                  "start": 49,
                  "end": 51,
                  "name": "hi"
                }
              ],
              "optional": false
            }
          }
        ]
      }
    }
  ],
  "sourceType": "module"
}
```

AST将标识符（identifier）、表达式（expression）、语句（statement）等抽象成为树状结构。

## 实现一个简单的 parser

![image-20210321210154324](https://technologybook.tech/assets/img/ast3.png)

Lever.ts

```typescript
import {IHelper} from "./helper";

namespace Checker {
    export function isBrackets(ch: string) {
        return /[\(\)\{\}\[\]\,\;]/i.test(ch);
    }
    export function isWhitespace(ch: string) {
        return " \t\n".indexOf(ch) >= 0;
    }
    export function isNumber(ch: string) {
        return /[0-9]/i.test(ch);
    }
    export function isVarName(ch: string) {
        return /\w+|_|$/i.test(ch);
    }
    export function isStartString(ch: string) {
        return '"' === ch;
    }
    export function isOp(ch: string) {
        return /[\+\-\*\/\=\&\|\!\<\>]/i.test(ch);
    }
}

export default class Lexer implements IHelper<{type: string, value: string|number}>{
    currentPosVal: {type: string, value: string|number} = {} as any

    current() {
        return this.currentPosVal || (this.currentPosVal = this.readByPos());
    }

    end() {
        return this.strHelper.end();
    }

    next() {
        let value = this.current();
        this.currentPosVal = null;
        return value;
    }

    constructor(public strHelper: IHelper<string>) {
        this.strHelper = strHelper;
    }

    pack(fn: (ch: string) => boolean) {
        let box = "";
        while (fn(this.strHelper.current()) && !this.strHelper.end()) {
            box += this.strHelper.next();
        }
        return box;
    }

    handleNumber() {
        const value = this.pack(Checker.isNumber);
        return {
            type: 'Number',
            value: parseInt(value)
        }
    }

    handleVar() {
        const value = this.pack(Checker.isVarName);
        return {
            type: "Variable",
            value: value
        }
    }

    handleString(){
        return {type: "String", value: this.readStringToEnd()}
    }

    handleWhitespace() {
        this.pack(Checker.isWhitespace)
    }

    readStringToEnd() {
        let isEnd = false,
            str = "";
        while (!this.strHelper.end()) {
            let c = this.strHelper.next();
            if (c === '"') {
                isEnd = true
            } else {
                str += c;
            }
        }
        return str;
    }

    private handleOperation() {
        {
            return {
                type: "Operation",
                value: this.pack(Checker.isOp)
            }
        }
    }

    private handleBrackets() {
        return {
            type: "Bracket",
            value: this.strHelper.next()
        };
    }

    readByPos() {
        this.handleWhitespace();
        if (this.strHelper.end()) return {type: "End", value: "End"};
        let c = this.strHelper.current();
        if (Checker.isStartString(c)) return this.handleString();
        if (Checker.isNumber(c)) return this.handleNumber()
        if (Checker.isVarName(c)) return this.handleVar()
        if (Checker.isBrackets(c)) return this.handleBrackets();
        if (Checker.isOp(c)) return this.handleOperation();
        return {type: "Error", value: "Error"};
    }
}

```

Parser.ts

```typescript
import Lexer from "./lexer";

function parser(token: Lexer) {
    const program = [];
    if (!token) return "";
    function isBracket(ch) {
        var tok = token.current();
        return tok && tok.type == "Bracket" && (!ch || tok.value == ch) && tok;
    }
    function isKeywords(kw) {
        var tok = token.current();
        return tok && tok.type == "KeyWord" && (!kw || tok.value == kw) && tok;
    }
    function isOp(op?) {
        var tok = token.current();
        return tok && tok.type == "Operation" && (!op || tok.value == op) && tok;
    }
    function skipBracket(ch) {
        if (isBracket(ch)) token.next();
    }
    function skipKeywords(kw) {
        if (isKeywords(kw)) token.next();
    }
    function skipOp(op) {
        if (isOp(op)) token.next();
    }
    function parseBlock() {
        var body = [];
        skipBracket("{");

        while (!isBracket("}")) {
            var sts = parse()
            sts && body.push(sts);
        }
        skipBracket("}");
        return {
            type: "BlockStatement",
            body: body
        }
    }
    function parseVariable() {
        skipKeywords("var");
        return {
            type  : "VariableDeclaration",
            declarations: parseDeclarator(),
            kind  : "var"
        }
    }

    function parseDeclarator() {
        var id = parseIdentifier();
        skipOp("=");
        return {
            type: "VariableDeclarator",
            id: id,
            init: parseExpression()
        }
    }


    function parseExpression() {
        return maybeCall(function(){
            return maybeBinary(parseAtom(), 0);
        });
    }

    function maybeBinary(left, le) {
        var tok = isOp();
        if (tok) {
            var prec = {
                "=": 1,
                "||": 2,
                "&&": 3,
                "<": 7,
                ">": 7,
                "<=": 7,
                ">=": 7,
                "==": 7,
                "!=": 7,
                "+": 10,
                "-": 10,
                "*": 20,
                "/": 20,
                "%": 20,
            }[tok.value];
            if (prec > le) {
                token.next();
                return maybeBinary({
                    type     : tok.value == "=" ? "assign" : "binary",
                    operator : tok.value,
                    left     : left,
                    right    : maybeBinary(parseAtom(), prec)
                }, le);
            }
        }
        return left;
    }

    function parseAtom() {
        return maybeCall(function(){
            if (isBracket("(")) {
                token.next();
                var exp = parseExpression();
  

    function parse() {
        if(isBracket(";")) skipBracket(";");
        else if (isBracket("{")) return parseBlock();
        else if (isKeywords("var")) return parseVariable();
        else if (isKeywords("if")) return parseIf();
        else return parseExpression();
    }

    while (!token.end()) {
        program.push(parse())
    }
    return {type: "Program", value: []}
}
```

