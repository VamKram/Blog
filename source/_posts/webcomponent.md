title: Typescript Template Literal Types
date: 2020/01/28
cover: https://technologybook.tech/assets/img/wc.png
categories:
- html
tags:
- webComponent

---
# webcomponent

> 我始终认为 ~~webcomponent有着巨大的潜力，就像PWA注定是未来~~。

### 什么事web compoennt

> Web Components旨在解决这些问题 — 它由三项主要技术组成，它们可以一起使用来创建封装功能的定制元素，可以在你喜欢的任何地方重用，不必担心代码冲突。
>
> - **Custom elements（自定义元素）：**一组JavaScript API，允许您定义custom elements及其行为，然后可以在您的用户界面中按照需要使用它们。
> - **Shadow DOM（影子DOM）**：一组JavaScript API，用于将封装的“影子”DOM树附加到元素（与主文档DOM分开呈现）并控制其关联的功能。通过这种方式，您可以保持元素的功能私有，这样它们就可以被脚本化和样式化，而不用担心与文档的其他部分发生冲突。
> - **HTML templates（HTML模板）：** [`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/template) 和 [``](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/slot) 元素使您可以编写不在呈现页面中显示的标记模板。然后它们可以作为自定义元素结构的基础被多次重用。
>
> from mdn

## APIS

- **`customElements`**

  > **`customElements`** 是[`Window`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window)对象上的一个只读属性，接口返回一个[`CustomElementRegistry`](https://developer.mozilla.org/zh-CN/docs/Web/API/CustomElementRegistry) 对象的引用，可用于注册新的 [custom elements](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements)，或者获取之前定义过的自定义元素的信息。

  ```javascript
  
  
  class Test extends HTMLElement {
    constructor() {
      super();
  
      const content = document.createElement('div');
      const p = document.createElement('p');
      p.innerText = 'Hello';
      content.append(p);
      // this => Html element
      this.append(content);
    }
  }
  
  window.customElements.define('test', Test);
  ```

  [生命周期回调](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements#Using_the_lifecycle_callbacks)

  定义在自定义元素的类定义中的特殊回调函数，影响其行为：

  - `connectedCallback`: 当自定义元素第一次被连接到文档DOM时被调用。
  - `disconnectedCallback`: 当自定义元素与文档DOM断开连接时被调用。
  - `adoptedCallback`: 当自定义元素被移动到新文档时被调用。
  - `attributeChangedCallback`: 当自定义元素的一个属性被增加、移除或更改时被调用。

  ```javascript
  attributeChangedCallback(attributeName, oldValue, newValue) {
      if (attributeName==="") {
          console.log(oldValue, newValue)
      }
  }
  ```

  

- ## Template标签

  如果使用template 上述代码的写法应该是

```html

<template id="test">
  <div class="content">
    <p>Hello</p>
  </div>
</template>
```

```javascript

class UserCard extends HTMLElement {
  constructor() {
    super();
		const shadow = this.attachShadow( { mode: 'closed' } );
    const t = document.getElementById('test');
    const content = t.content.cloneNode(true);
    this.appendChild(content);
  }
}
```

- ## Shadow DOM

  Web Component 允许内部代码隐藏起来，这叫做 Shadow DOM，即这部分 DOM 默认与外部 DOM 隔离，内部任何代码都无法影响外部。

  自定义元素的`this.attachShadow()`方法开启 Shadow DOM，详见下面的代码。

  当想添加事件给shadow的时候

  ```javascript
  
  p = shadow.querySelector('p');
  ```

  

## 一个例子

```javascript
<template>
    <style>
        .coloured {
            color: red;
        }
    </style>
    <p>the first webcompnent is  <strong class="coloured">Hello World</strong></p>
</template>
<script>
    (function() {
        // Creates an object based in the HTML Element prototype
        // 基于HTML Element prototype 创建obj
        var element = Object.create(HTMLElement.prototype);
        // 获取特mplate的内容
        var template = document.currentScript.ownerDocument.querySelector('template').content;
        // element创建完成之后的回调
        element.createdCallback = function() {
            // 创建 shadow root
            var shadowRoot = this.createShadowRoot();
            // 向root中加入模板
            var clone = document.importNode(template, true);
            shadowRoot.appendChild(clone);
        };
        document.registerElement('hellow-world', {
            prototype: element
        });
    }());
</script>
```



