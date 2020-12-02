---
title: 垒积木之实现一个MiniReact
date: 2017/12/05
cover: https://technologybook.tech/assets/img/react.png
categories:
- react
tags: 
- react source

---

![rs](https://images.pexels.com/photos/1056555/pexels-photo-1056555.jpeg?cs=srgb&dl=abstract-arms-art-1056555.jpg&fm=jpg)
# 垒积木之实现一个MiniReact

> 最近在学习React source code， 一点一点的总算是啃完了，于是也想实现一个React的lib，不求事无巨细的实现框架的每一个细节，希望能体现React的精髓帮助学习。

### 以一个hello world开始实现virtual dom

如果我们写一个React的App，代码可能是：

```javascript
class Hello extends Component {
    render() {
        return (
        	<p>hello world</p>
        )
    }
}

ReactDOM.render(<Hello />, document.getElementById('root'));

// JSX 最后会被编译成形如

ReactDOM.render(
  React.createElement('p', null, 'Hello World'),
  document.getElementById('root')
);

```

所以，一个React Component 起点就是createElement方法，

```javascript
function createVDOM(type, props, contArg) {
    let content, args;
    content = contArg || "";
    // 生成抽象DOM结构。
    return {
        vDom: true,
        type,
       	key: props.key || null,
        props: props || {},
        content: [{text: content}]
    }
}

// render method

function render(vDOM) {
    const node;
    if(vDOM.text) {
        node = document.createTextNode(vDOM.text);
    } else if (vDOM.vDom) {
        let {vDom, type, key, props, content} = vDOM;
        
        if(typeof type === 'string') {
             node = document.createElement(type);
        }
        if(_.isArray("content")){
            appendChildren(node, render(content));
        }
    }
}


// appendChildren 方法插入节点
function appendChildren(
  parent,
  children,
  start = 0,
  end = children.length - 1,
  beforeNode
) {
  while (start <= end) {
    var ch = children[start++];
    parent.insertBefore(mount(ch), beforeNode);
  }
}

// 拥有了这三个基本的方法我们就可以完成一个，简单的hello world

var vDOM = createVDOM('p', null, 'hello world');
document.getElelment('root').appendChild(render(vDOM));

```



### 增加 Props的支持

```javascript
function createVDOM(type, props, contArg) {
    let content, args;
    // 生成抽象DOM结构。
    if (typeof type !== 'string') {
        // 处理嵌套逻辑
    } else {
        content = [{ _text: contArg == null ? "" : contArg }];
    }
    return {
        vDom: true,
        type,
       	key: props.key || null,
        props: props || {},
        content
    }
}

// render method

function render(vDOM) {
    const node;
    if(vDOM.text) {
        node = document.createTextNode(vDOM.text);
    } else if (vDOM.vDom) {
        let {vDom, type, key, props, content} = vDOM;
        
        if(typeof type === 'string') {
             node = document.createElement(type);
             if(_.isArray("content")){
            appendChildren(node, render(content));
        	}
        } else {
            var vnode = type(props, content);
            node = mount(vnode);
            vDOM.text = vnode;
        }
       
    }
    return node;
}


// appendChildren 方法插入节点
function appendChildren(
  parent,
  children,
  start = 0,
  end = children.length - 1,
  beforeNode
) {
  while (start <= end) {
    var ch = children[start++];
    parent.insertBefore(mount(ch), beforeNode);
  }
}
// 通过对function的分流，可以讲props以回调的形式交给render函数处理

var vDOM = (props) => createVDOM('p', null, `hello ${props.name}`);
var div = createVDOM(vDOM, {name: 'mark'})
document.getElelment('root').appendChild(render(div));
```



### 添加多节点的处理

```javascript
function createVDOM(type, props, contArg) {
    let content, args;
    // contArg参数的数量，做判断
    args = arguments - 2;
    // 生成抽象DOM结构。
    if (typeof type !== 'string') {
        // 处理嵌套逻辑
        if(args == 1){
            content = contArg;
        }
    } else {
        if (args > 1) {
            args = Array(len);
            for (i = 0; i < len; i++) {
              args[i] = arguments[i + 2];
            }
            content = maybeFlatten(args, false);
        }
        
        if (args < 1) {
            content = {};
        }
        content = [{ _text: contArg == null ? "" : contArg }];
    }
    return {
        vDom: true,
        type,
       	key: props.key || null,
        props: props || {},
        content
    }
}

//压平
function maybeFlatten(arr) {
    for (var i = 0; i < arr.length; i++) {
      var ch = arr[i];
   	if (!ch.vDom) {
      arr[i] = { _text: ch == null ? "" : ch };
    }
  }
  return arr;
}

// render method

function render(vDOM) {
    const node;
    if(vDOM.text) {
        node = document.createTextNode(vDOM.text);
    } else if (vDOM.vDom) {
        let {vDom, type, key, props, content} = vDOM;
        
        if(typeof type === 'string') {
             node = document.createElement(type);
             if(_.isArray("content")){
            appendChildren(node, render(content));
        	}
        } else {
            var vnode = type(props, content);
            node = mount(vnode);
            vDOM.text = vnode;
        }
       
    }
    return node;
}


// appendChildren 方法插入节点
function appendChildren(
  parent,
  children,
  start = 0,
  end = children.length - 1,
  beforeNode
) {
  while (start <= end) {
    var ch = children[start++];
    parent.insertBefore(mount(ch), beforeNode);
  }
}
// 通过对function的分流，可以讲props以回调的形式交给render函数处理

// 支持参数数组多节点
function Box(props, content) {
      return createVDOM('div', null, createVDOM('h1', null, props.title), createVDOM('p', null, content));
    }

    const vnode = createVDOM('div', null, createVDOM(Box, { title: 'Fancy Box' }, 'content'));
document.getElelment('root').appendChild(render(vnode));

```

到此，React的VDOM产生真实DOM的整个过程已经可以粗糙的模拟出来，对VDOM的了解也更进一步。