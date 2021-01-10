# Jsx

- `Jsx`与`fiber节点`是同一个东西吗？

- `React Component`,`React Element`是同一个东西吗，和`Jsx`有什么关系

`jsx`在编译时会被`babel`编译为`React.createElement`方法

这就是为什么在每一个`Jsx`的 js 文件中，必须显示声明

```javascript
import React from "react";
```

# React.createElement 做了什么？

```javascript
export function createElement(type, config, children) {
  let propName;

  const props = {};

  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  if (config != null) {
    // 将 config 处理后赋值给 props
    // ...省略
  }

  const childrenLength = arguments.length - 2;
  // 处理 children，会被赋值给props.children
  // ...省略

  // 处理 defaultProps
  // ...省略

  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props
  );
}

const ReactElement = function (type, key, ref, self, source, owner, props) {
  const element = {
    // 标记这是个 React Element
    $$typeof: REACT_ELEMENT_TYPE,

    type: type,
    key: key,
    ref: ref,
    props: props,
    _owner: owner,
  };

  return element;
};
```

> 最终会调用`ReactElement`方法返回一个包含组件数据的对象，对象有个参数`$$typeof:REACT_ELEMENT_TYPE`标记该对象是一个`React Element`

额外`React`还提供了一个验证合法`React Element`对象的全局 Api`React.isVaildElement`

```javascript
export function isValidElement(object) {
  return (
    typeof object === "object" &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}
```

换言之，在`React`中所有，`jsx`在运行中的返回结果都是`React Element`

# React Component

```javascript
class AppClass extends React.Component {
  render() {
    return <p>KaSong</p>;
  }
}
console.log("这是ClassComponent：", AppClass);
console.log("这是Element：", <AppClass />);

function AppFunc() {
  return <p>KaSong</p>;
}
console.log("这是FunctionComponent：", AppFunc);
console.log("这是Element：", <AppFunc />);
```

可以看出`ClassComponent`对应的`Element`的`type`字段为`AppClass`自身
，`fnComponent`也一样

```javascript
{
  $$typeof: Symbol(react.element),
  key: null,
  props: {},
  ref: null,
  type: ƒ AppFunc(),
  _owner: null,
  _store: {validated: false},
  _self: null,
  _source: null
}

```

# Jsx 与 fiber 节点

`jsx`是一个描述当前组件的数据结构，不包含组件`Scheduler`,
`Reconciler`,`Renderer`所需的相关信息

以下信息不包含在`jsx`中

- 组件的更新优先级
- state
- 被打上的`Renderer`标记

所以在组件`mount`时，`Reconciler`根据`jsx`描述的组件内容生成组件对应的`fiber`节点
在`update`时，`Reconciler`将`jsx`与`fiber`保存的数据对比，生成新的组件对应的`fiber节点`，并将对比结果打上`tag`
