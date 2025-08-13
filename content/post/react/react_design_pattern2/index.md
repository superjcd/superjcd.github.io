---
title: "React常用设计模式(新)"
description: 
date: 2025-08-13T07:45:54Z
categories:
    - web开发
tags:
    - react
weight: 1 
---
## Compound pattern

这个模式的它能解决怎样的问题? 其实没有解决什么问题，只是把需要共用相同state的组件用一种合理的方式组合在一起罢了。

我们先看一下使用了Compound模式的组件代码：

```tsx
import React from "react";

import { FlyOut } from "./FlyOut";



export default function FlyoutMenu() {

  return (

    <FlyOut>

      <FlyOut.Toggle />

      <FlyOut.List>

        <FlyOut.Item>Edit</FlyOut.Item>

        <FlyOut.Item>Delete</FlyOut.Item>

      </FlyOut.List>

    </FlyOut>

  );

}
```

> 这个模式出现在很多地方，比如Shadcn中的组件，也大多是基于多个更小的组件组合起来的。 不一定非得使用Flyout.xxxx来表示子组件, 但是这种命名方式更加清晰

Toggle，List， Item组件都复用了Flyout的状态value。

Flyout的定义：

```tsx
const FlyOutContext = createContext();



function Flyout({children}) {

  const [value, setValue] = useState(false)



  <FlyOutContext.Provider value={{value, setValue}}>

    {chirdren}

  </FlyOutContext.Provider>

}
```

然后以Toggle组件为例：

```tsx
Flyout.Toggle = ({chidren}) => {

  const {value, setValue} = useContext(FlyOutContext)



  return  (

     <div onClick={() => setVale(!value)}>

      <Icon />

    </div>

  )

}
```

其他的组件也是用一同样的方式获取到value， 这个不赘述了；这里需要追加的一点的是， 给多个子元素传递状态的方式除了上面这种使用Context， 也可以直接通过遍历React.Children结合React.cloneElement函数来实现, 比如：

```tsx
export function FlyOut(chilren) {

  const [value, setValue] = React.useState(false);



  return (

    <div>

      {React.Children.map(children, child =>

        React.cloneElement(child, { value, setValue })

      )}

    </div>

  );

}
```

## HOC （High-order component）

### 基础示例

```typescript
import React from 'react';



const withAsync = (WrappedComponent, LoadingComponent = () => <div>Loading...</div>) => {

  return (props) => {

    if (props.loading) {

      return <LoadingComponent {...props} />;

    }

    return <WrappedComponent {...props} />;

  };

};



export default withAsync;
```

上面展示了一个基础的HOC示例，withAsny函数会根据props的loading属性来决定是不是要添加一个等待的过渡效果, 好处在于逻辑复用(类似于装饰器函数)

### 带选项options

```jsx
import React from 'react';



const withAsync = (WrappedComponent, options = {}) => {

  const {

    LoadingComponent = () => <div>Loading...</div>,

    ErrorComponent = ({ error }) => <div>Error: {error}</div>,

    loadingPropName = 'loading',

    errorPropName = 'error'

  } = options;



  return (props) => {

    if (props[errorPropName]) {

      return <ErrorComponent error={props[errorPropName]} {...props} />;

    }

    if (props[loadingPropName]) {

      return <LoadingComponent {...props} />;

    }

    return <WrappedComponent {...props} />;

  };

};



export default withAsync;
```

带options的HOC， 可以类比为带参数的装饰器函数, 可以更加灵活地处理不同的应用场景

使用场景：

```typescript
function DataDisplay({ data }) {

  return <div>{JSON.stringify(data, null, 2)}</div>;

}



const EnhancedDataDisplay = withAsync(DataDisplay, {

  LoadingComponent: Spinner,

  ErrorComponent: ErrorMessage,

  loadingPropName: 'isFetching',

  errorPropName: 'fetchError'

});




  ...



  return (

    <div>

      <h1>Data Display</h1>

      <EnhancedDataDisplay

        data={state.data}

        isFetching={state.isFetching}

        fetchError={state.fetchError}

      />

    </div>

  );
```

typescipt 版本：

```tsx
import React from 'react';



interface WithAsyncOptions<P = any> {

  LoadingComponent?: React.ComponentType<P>;

  ErrorComponent?: React.ComponentType<P & { error: any }>;

  loadingPropName?: string;

  errorPropName?: string;

}



function withAsync<P extends object>(

  WrappedComponent: React.ComponentType<P>,

  options: WithAsyncOptions<P> = {}

): React.FC<P> {

  const {

    LoadingComponent = () => <div>Loading...</div>,

    ErrorComponent = ({ error }) => <div>Error: {error}</div>,

    loadingPropName = 'loading',

    errorPropName = 'error'

  } = options;



  return (props: P & { [key: string]: any }) => {

    if (props[errorPropName]) {

      return <ErrorComponent error={props[errorPropName]} {...props} />;

    }

    if (props[loadingPropName]) {

      return <LoadingComponent {...props} />;

    }

    return <WrappedComponent {...props} />;

  };

}



export default withAsync;
```

### 组合多个HOC情况

compose函数(用来结合多个HOC):

```tsx
const compose = (...hocs) => (Component) =>

  hocs.reduceRight(

    (WrappedComponent, hoc) => hoc(WrappedComponent),

    Component

  );
```

使用compose场景(结合context)

```tsx
const withUserContext = (WrappedComponent) => {

  return (props) => (

    <UserContext.Consumer>

      {(user) => <WrappedComponent {...props} user={user} />}

    </UserContext.Consumer>

  );

};



const EnhancedComponent = compose(

  withTheme,

  withLoading,

  withUserContext

)(MyComponent);
```

### Render props替代HOC场景

譬如实现基于登录状态的条件渲染：

```tsx
function Auth({ children }) {

  const [user, setUser] = useState(null);



  useEffect(() => {

    checkAuth().then(setUser);

  }, []);



  return children({

    isAuthenticated: !!user,

    user

  });

}



// 使用

<Auth>

  {({ isAuthenticated, user }) =>

    isAuthenticated

      ? <Dashboard user={user} />

      : <Login />

  }

</Auth>
```

但是这里还是需要注意的一点在于： 和前面的HOC的条件渲染不同， 这里的条件渲染事实上是在子组件实现的(前面的HOC是在Wrapper函数实现的)， 外部的Auth函数只是提供给Children必要的props而已； 但是可不可以把UI逻辑放到Auth组件里？其实也是可以的；但是render props模式一般只是提供给children共享的状态(比如上面的user)；  

另外假设有多个子组件要共用状态的话， 可以考虑：

```jsx
<Auth>

  {({ isAuthenticated, user }) => (

    <>

      {isAuthenticated ? (

        <>

          <Dashboard user={user} />

          <Profile user={user} />

        </>

      ) : (

        <>

          <Login />

          <Signup />

        </>

      )}

    </>

  )}

</Auth>
```

或者直接使用Context(诸如前面的Compund模式)， context适合更加复杂（比如深度嵌套）的状态共享

## Props Collection 模式

### 基础用法

鼠标交互逻辑

```jsx
function useMouse() {

  const [position, setPosition] = useState({ x: 0, y: 0 });

  const mouseProps = {

    onMouseMove: (e) => setPosition({ x: e.clientX, y: e.clientY }),

    style: { cursor: 'pointer' },

  };

  return {

    position,

    mouseProps,

  };

}



function MouseTracker() {

  const { position, mouseProps } = useMouse();

  return (

    <div {...mouseProps}>

      Mouse position: {position.x}, {position.y}

    </div>

  );

}
```

简单来讲Props collection就是把相关的props集合在了一起， 比如上面拿到useMouse, 把位置相关的， 以及和鼠标位置相关联的action组合在了一起

### 结合Render Props模式

```jsx
function Tooltip({ children, content }) {

  const [isVisible, setIsVisible] = useState(false);

  const tooltipProps = {

    onMouseEnter: () => setIsVisible(true),

    onMouseLeave: () => setIsVisible(false),

  };

  return children({

    isVisible,

    tooltipProps,

  });

}



function App() {

  return (

    <Tooltip content="This is a tooltip">

      {({ isVisible, tooltipProps }) => (

        <div>

          <button {...tooltipProps}>Hover me</button>

          {isVisible && <div className="tooltip">Tooltip content</div>}

        </div>

      )}

    </Tooltip>

  );

}
```

### 使用多种Props集合

```jsx
function useHover() {

  const [isHovered, setIsHovered] = useState(false);

  const hoverProps = {

    onMouseEnter: () => setIsHovered(true),

    onMouseLeave: () => setIsHovered(false),

  };

  return {

    isHovered,

    hoverProps,

  };

}



function useFocus() {

  const [isFocused, setIsFocused] = useState(false);

  const focusProps = {

    onFocus: () => setIsFocused(true),

    onBlur: () => setIsFocused(false),

  };

  return {

    isFocused,

    focusProps,

  };

}



function Button() {

  const { isHovered, hoverProps } = useHover();

  const { isFocused, focusProps } = useFocus();

  return (

    <button

      {...hoverProps}

      {...focusProps}

      style={{

        backgroundColor: isHovered ? 'lightblue' : 'white',

        borderColor: isFocused ? 'blue' : 'gray',

      }}

    >

      Interactive Button

    </button>

  );

}
```

### Props Getter 模式

不直接返回Props对象， 而是说利用闭包返回可以生成Props的函数， 这样在后续使用中可以更加灵活

```jsx
function useToggle(initialOn = false) {

  const [on, setOn] = useState(initialOn);

  const toggle = () => setOn(!on);

  // Props Getter 函数

  const getTogglerProps = ({ onClick, ...props } = {}) => ({

    'aria-pressed': on,

    onClick: () => {

      toggle();

      onClick?.();

    },

    ...props,

  });

  return {

    on,

    toggle,

    getTogglerProps,

  };

}



function App() {

  const { on, getTogglerProps } = useToggle();

  return (

    <div>

      <button

        {...getTogglerProps({

          onClick: () => console.log('Button clicked'),

          className: 'my-button',

        })}

      >

        {on ? 'ON' : 'OFF'}

      </button>

    </div>

  );

}
```

## State Reducer模式

State Reducer 模式的核心思想是：​**​组件仍然管理自己的状态，但将状态更新的决定权通过 reducer 函数暴露给使用者​**​

### 传统Reducer

```jsx
function toggleReducer(state, action) {

  switch (action.type) {

    case 'TOGGLE':

      return { on: !state.on };

    default:

      return state;

  }

}



function useToggle() {

  const [state, dispatch] = useReducer(toggleReducer, { on: false });

  const toggle = () => dispatch({ type: 'TOGGLE' });

  return {

    on: state.on,

    toggle,

  };

}
```

### 添加State Reducer

```jsx
function useToggle({ reducer = toggleReducer } = {}) {

  const [state, dispatch] = useReducer(reducer, { on: false });

  const toggle = () => dispatch({ type: 'TOGGLE' });

  return {

    on: state.on,

    toggle,

  };

}



// 使用默认行为

function App() {

  const { on, toggle } = useToggle();

  return <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>;

}



// 自定义 reducer

function App() {

  const { on, toggle } = useToggle({

    reducer(state, action) {

      if (action.type === 'TOGGLE' && on) {

        return state; // 阻止关闭

      }

      return toggleReducer(state, action);

    }

  });

  return <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>;

}
```

### 使用场景

```jsx
function useForm({ reducer = formReducer } = {}) {

  const [state, dispatch] = useReducer(reducer, { values: {} });

  const setFieldValue = (name, value) =>

    dispatch({ type: 'SET_FIELD', name, value });

  return {

    values: state.values,

    setFieldValue,

  };

}



// 自定义 reducer 添加日志

function LoggingForm() {

  const { values, setFieldValue } = useForm({

    reducer(state, action) {

      console.log('Action:', action);

      const newState = formReducer(state, action);

      console.log('New state:', newState);

      return newState;

    }

  });

  // 渲染逻辑...

}
```

State Reducer的实现基础在于 reducer函数签名的一致性：（state, action） => void  

也正因为多个reducer共享相同的函数签名， 所以State Reducer也可以和组合多个HOC那样实现composer模式(类似前面组合多个HOC一样)

### 多个reducer组合

```jsx
function composeReducers(...reducers) {

  return (state, action) =>

    reducers.reduce((currentState, reducer) =>

      reducer(currentState, action), state);

}



function useAdvancedToggle() {

  const { on, toggle } = useToggle({

    reducer: composeReducers(

      toggleReducer,

      (state, action) => {

        if (action.type === 'TOGGLE' && state.on) {

          // 添加额外限制

          return state;

        }

        return state;

      },

      (state, action) => {

        // 添加日志

        console.log(action);

        return state;

      }

    )

  });

}
```

## 总结

其实上面的设计模式， 核心就是围绕  

- 关注点分离， 比如Container & presention模式(比较简单，上面没有详细展开)以及Props render，  Props collections模式都是如此  

- 逻辑复用， 比如HOC以及State Reducer;  前提是可复用逻辑本身和目标是存在正交性的；当然严格以上来说逻辑复用其实也是关注点分离的一种表现形式， 把重复的逻辑和主逻辑抽离出来  

上诉react设计模式之所以需要强调**关注点分离**的一个重要原因在于：React Hook本身在设计上， 将UI和逻辑紧密地糅在了一起（相对的Vue3将逻辑也就是script和UI进行了分离， 两种流派各有利弊），所以在设计复杂应用的时候又需要以某种方式实现关注点分离