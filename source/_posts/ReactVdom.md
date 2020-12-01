---
title: React 系列（virtual DOM）
date: 2017/11/19
categories:
- react
tags: 
- react source

---

# React 系列（virtual DOM）

> 一直以来，vDom以及react的diff算法都被认为是react性能的保障，二者相辅相成，构建高性能react base app发挥着作用，为了更深入的对react的了解，遂对react实现方式进行一次探究

	VDom之所以存在的原因，是由于传统的直接操作DOM的方式过于低效。根据浏览器的工作方式，操作一个DOM的过程如下:

### DOM render process

+ render engine 会解析HTML文件，生成DOM tree
+ render style 创建 解释样式的 render tree
+ reflow 根据renderer对象，分配一组屏幕坐标值
+ painting 调用处理UI的API吊起GUI线程完成绘制

而操作一个Vdom的过程则是：

### Virtual DOM render process

+ 对DOM信息进行抽象
+ 修改节点时不会每次都重绘，把多次对Vdom的操作整合
+ 调用DOM API （createElement， inserBfore等等。。）
+ DOM render process

可以看出Vdom是由于真实DOM操作成本过高的一种解决方案，Vdom是对dom的一种抽象，这个过程大大的优化了频繁操作DOM的成本。

#### Vdom的模型

> 简而言之Vdom是对real DOM 的抽象

一个VDOM需要具备描述节点类型，传递节点信息，传递子节点信息的能力。

```javascript
// ReactElement描述的节点信息
{
    $$typeof: REACT_ELEMENT_TYPE,
    type: type,
    key: key,
    ref: ref,
    props: props,
    _owner: owner,
  }
```

通过createElement创建Vdom的过程
![vd](https://camo.githubusercontent.com/ad7c297c15083efd21c946cc23bd064a083d11be/68747470733a2f2f646f63732e676f6f676c652e636f6d2f64726177696e67732f642f31317567425477446b716e3670326e35466b7073317033456c70385a546f49527a587a764d344c4a4d5961552f7075623f773d35343326683d323239)

```javascript
// react source code 
export function createElement(type, config, children) {
  let propName;

  // Reserved names are extracted
  const props = {};

  let key = null;
  let ref = null;
  let self = null;
  let source = null;
// 省略。。。
  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props,
  );
}
    
// reactElement 创建
const ReactElement = function(type, key, ref, self, source, owner, props) {
  const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner,
  };
    // ...
```

​	react 会根据组件的Type来判断，组件是文字节点，标签，或是自定义组件，然后通过递归组件type并调用DOMAPI，createElement方法或createText生成real dom，最终返回一个真正意义上的DOM。

在得到了一个Vdom的情况下需要进行vdom -> dom的转化，这个过程在源码中：

```javascript
// 在新架构之前ReactElement会调用一个instantiateReactComponent方法去处理相关逻辑
// 在最新的版本中createElement$1负责构建DOM tree
function createElement$1(type, props, rootContainerElement, parentNamespace) {
  var isCustomComponentTag = void 0;
// 在父容器的命名空间中创建标记
// 标签没有命名空间。
  var ownerDocument = getOwnerDocumentFromRootContainer(rootContainerElement);
  var domElement = void 0;
  var namespaceURI = parentNamespace;
  if (namespaceURI === HTML_NAMESPACE) {
    namespaceURI = getIntrinsicNamespace(type);
  }
  if (namespaceURI === HTML_NAMESPACE) {
    {
      isCustomComponentTag = isCustomComponent(type, props);
      // 判断是否是自定义Component或命名规范 
      !(isCustomComponentTag || type === type.toLowerCase()) ? warning(false, '<%s /> is using incorrect casing. ' + 'Use PascalCase for React components, ' + 'or lowercase for HTML elements.', type) : void 0;
    }

    if (type === 'script') {
      var div = ownerDocument.createElement('div');
      div.innerHTML = '<script><' + '/script>'; // eslint-disable-line
      // 确保产生一个script
      var firstChild = div.firstChild;
      domElement = div.removeChild(firstChild);
    } else if (typeof props.is === 'string') {
        // 根据Type创建标签
      domElement = ownerDocument.createElement(type, { is: props.is });
    } else {
      domElement = ownerDocument.createElement(type);
    }
  } else {
    domElement = ownerDocument.createElementNS(namespaceURI, type);
  }

  {
    if (namespaceURI === HTML_NAMESPACE) {
      if (!isCustomComponentTag && Object.prototype.toString.call(domElement) === '[object HTMLUnknownElement]' && !Object.prototype.hasOwnProperty.call(warnedUnknownTags, type)) {
        warnedUnknownTags[type] = true;
        warning(false, 'The tag <%s> is unrecognized in this browser. ' + 'If you meant to render a React component, start its name with ' + 'an uppercase letter.', type);
      }
    }
  }

  return domElement;
}

```

包括生成Text的方法，以及跳出递归的条件

```
// 生成TextNode的源码
function createTextNode$1(text, rootContainerElement) {
  return getOwnerDocumentFromRootContainer(rootContainerElement).createTextNode(text);
}
function getOwnerDocumentFromRootContainer(rootContainerElement) {
  return rootContainerElement.nodeType === DOCUMENT_NODE ? rootContainerElement : rootContainerElement.ownerDocument;
}
```

Text作为React递归非错误的跳出条件，逻辑很简单。

```
function batchedUpdates（fn，bookkeeping）{
   if（isBatching）{
     //在restore state之前完全完成时，等待batching。
     return fn（bookkeeping）;
  }
   isBatching = true;
   try{
     return _batchedUpdates（fn，bookkeeping）;
   } finally {
     //等到所有更新都传播完毕
     //恢复受控组件的状态。
     isBatching = false;
     var controlledComponentsHavePendingUpdates = needsStateRestore（）;
     if（controlledComponentsHavePendingUpdates）{
       //如果触发了受控事件，需要恢复状态
       // DOM节点返回受控值。
       //确保没有操作DOM的情况。
      _flushInteractiveUpdates（）;
      restoreStateIfNeeded（）;
    }
  }
}
```

至此react 从Vdom到dom的创建过程已经明了，当然React还包含了一套事件绑定机制，和充分利用rest 线程的fiber架构等等。