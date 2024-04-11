title: Typescript Template Literal Types
date: 2020/11/12
cover: https://technologybook.tech/assets/img/t.png
categories:
- ts
tags:
- ts

---
# Typescript Template Literal Types

> String literal types in TypeScript allow us to model functions and APIs that expect a set of specific strings.

### 先看一下基本用法

```typescript
type Color = "red" | "blue";
type Quantity = "one" | "two";

type SeussFish = `${Quantity | Color} fish`;
//   ^ = type SeussFish = "one fish" | "two fish" | "red fish" | "blue fish"
```

TLT提供了一种在字符串中添加变量的能力，这给typescript提供了极大的想象力。

```typescript
type VerticalAlignment = "top" | "middle" | "bottom";
type HorizontalAlignment = "left" | "center" | "right";

// Takes
//   | "top-left"    | "top-center"    | "top-right"
//   | "middle-left" | "middle-center" | "middle-right"
//   | "bottom-left" | "bottom-center" | "bottom-right"

declare function setAlignment(value: `${VerticalAlignment}-${HorizontalAlignment}`): void;

setAlignment("top-left");   // works!
setAlignment("top-middel"); // error!
Argument of type '"top-middel"' is not assignable to parameter of type '"top-left" | "top-center" | "top-right" | "middle-left" | "middle-center" | "middle-right" | "bottom-left" | "bottom-center" | "bottom-right"'.
setAlignment("top-pot");    // error! but good doughnuts if you're ever in Seattle
Argument of type '"top-pot"' is not assignable to parameter of type '"top-left" | "top-center" | "top-right" | "middle-left" | "middle-center" | "middle-right" | "bottom-left" | "bottom-center" | "bottom-right"'.
```

对于tlt的使用，大家都已经玩出花了，诸如template parser的有趣的实现。

## 这是一个提取params

```typescript
type ExtractRouteParams<T extends string> =
  string extends T
  ? Record<string, string>
  : T extends `${infer Start}:${infer Param}/${infer Rest}`
  ? {[k in Param | keyof ExtractRouteParams<Rest>]: string}
  : T extends `${infer Start}:${infer Param}`
  ? {[k in Param]: string}
  : {};


```



## 一个JSON Parser。。。

```typescript
type ParserError<T extends string> = { error: true } & T
type EatWhitespace<State extends string> =
  string extends State
    ? ParserError<"EatWhitespace got generic string type">
    : State extends ` ${infer State}` | `\n${infer State}`
      ? EatWhitespace<State>
      : State
type AddKeyValue<Memo extends Record<string, any>, Key extends string, Value extends any> =
  Memo & { [K in Key]: Value }
type ParseJsonObject<State extends string, Memo extends Record<string, any> = {}> =
  string extends State
    ? ParserError<"ParseJsonObject got generic string type">
    : EatWhitespace<State> extends `}${infer State}`
      ? [Memo, State]
      : EatWhitespace<State> extends `"${infer Key}"${infer State}`
        ? EatWhitespace<State> extends `:${infer State}`
          ? ParseJsonValue<State> extends [infer Value, `${infer State}`]
            ? EatWhitespace<State> extends `,${infer State}`
              ? ParseJsonObject<State, AddKeyValue<Memo, Key, Value>>
              : EatWhitespace<State> extends `}${infer State}`
                ? [AddKeyValue<Memo, Key, Value>, State]
                : ParserError<`ParseJsonObject received unexpected token: ${State}`>
            : ParserError<`ParseJsonValue returned unexpected value for: ${State}`>
          : ParserError<`ParseJsonObject received unexpected token: ${State}`>
        : ParserError<`ParseJsonObject received unexpected token: ${State}`>
type ParseJsonArray<State extends string, Memo extends any[] = []> =
  string extends State
    ? ParserError<"ParseJsonArray got generic string type">
    : EatWhitespace<State> extends `]${infer State}`
      ? [Memo, State]
      : ParseJsonValue<State> extends [infer Value, `${infer State}`]
        ? EatWhitespace<State> extends `,${infer State}`
          ? ParseJsonArray<EatWhitespace<State>, [...Memo, Value]>
          : EatWhitespace<State> extends `]${infer State}`
            ? [[...Memo, Value], State]
            : ParserError<`ParseJsonArray received unexpected token: ${State}`>
        : ParserError<`ParseJsonValue returned unexpected value for: ${State}`>
type ParseJsonValue<State extends string> =
  string extends State
    ? ParserError<"ParseJsonValue got generic string type">
    : EatWhitespace<State> extends `null${infer State}`
      ? [null, State]
      : EatWhitespace<State> extends `true${infer State}`
        ? [true, State]
        : EatWhitespace<State> extends `false${infer State}`
          ? [false, State]
          : EatWhitespace<State> extends `"${infer Value}"${infer State}`
            ? [Value, State]
            : EatWhitespace<State> extends `[${infer State}`
              ? ParseJsonArray<State>
              : EatWhitespace<State> extends `{${infer State}`
                ? ParseJsonObject<State>
                : ParserError<`ParseJsonValue received unexpected token: ${State}`>
export type ParseJson<T extends string> =
  ParseJsonValue<T> extends infer Result
    ? Result extends [infer Value, string]
      ? Value
      : Result extends ParserError<any>
        ? Result
        : ParserError<"ParseJsonValue returned unexpected Result">
    : ParserError<"ParseJsonValue returned uninferrable Result">
```



## 以及我的 extra url type

```typescript
type ParsePathname<T extends string> = T extends `${infer Left}?${infer Right}`
  ? Right extends `:${infer Rest}`
    ? `${Left}?:${ParsePathname<Rest>}`
    : Left
  : T
type ParseQueryString<T extends string> = T extends `${infer _Left}?${infer Right}`
  ? Right extends `:${infer Rest}`
    ? ParseQueryString<Rest>
    : Right
  : ''
export type ParseUrl<T extends string> = CleanEmptyObject<{
  pathname: string
  params: ParseData<ParsePathname<T>>
  query: ParseData<ParseQueryString<T>>
}>
```

以下是一些效果

![](/Users/user/Desktop/t1.png)

### 以及更多的[花式应用...](https://github.com/ghoullier/awesome-template-literal-types)