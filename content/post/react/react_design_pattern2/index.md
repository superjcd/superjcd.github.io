# 

---

title: React常用设计模式(二)
description: 
slug: 
date: 2025-08-12
categories:
    - web开发
tags:
    - react
weight: 1       

---

## React常用设计模式(二)

### HOC



#### 基础示例

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

上面展示了一个基础的HOC示例，withAsy函数会更具props的loading属性来决定是不是要添加一个等待的过渡效果, 好处在于逻辑复用(类似于装饰器函数)



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

### vue3中的HOC替代方案

不同于HOC在react中被频繁使用的请况，  在vue使用的很少；根本原因在于vue是把代码逻辑(script部分)和UI(template部分)进行了拆分, 而不像react那样进行混编；

所以在vue3中有分别针对UI和逻辑的高阶函数

- UI复用 - 基于插槽
  
  ```
  <!-- AsyncWrapper.vue -->
  <template>
    <slot v-if="loading" name="loading" />
    <slot v-else-if="error" name="error" :error="error" />
    <slot v-else :data="data" />
  </template>
  
  <script setup>
  import { ref, onMounted } from 'vue'
  
  const props = defineProps(['fetcher'])
  const data = ref(null)
  const loading = ref(true)
  const error = ref(null)
  
  onMounted(async () => {
    try {
      data.value = await props.fetcher()
    } catch (err) {
      error.value = err
    } finally {
      loading.value = false
    }
  })
  </script>
  
  <!-- 使用 -->
  <AsyncWrapper :fetcher="fetchData">
    <template #default="{ data }">
      <UserList :users="data" />
    </template>
    <template #loading>
      <Spinner 
  ```

- 组合式函数（逻辑复用）
  
  ```
  // useAsync.js
  import { ref } from 'vue'
  
  export function useAsync(asyncFn) {
    const data = ref(null)
    const loading = ref(false)
    const error = ref(null)
  
    const execute = async () => {
      try {
        loading.value = true
        data.value = await asyncFn()
      } catch (err) {
        error.value = err
      } finally {
        loading.value = false
      }
    }
  
    return { data, loading, error, execute }
  }
  
  // 组件中使用
  import { useAsync } from './useAsync'
  
  export default {
    setup() {
      const { data, loading, error } = useAsync(() => fetch('/api/data'))
      return { data, loading, error }
    }
  }
  ```

- 自定义指令(略)



#### 多个HOC复用的情况

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





#### Render props替代HOC场景

譬如实现条件渲染：

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

但是这里还是需要注意的一点在于： 和前面的HOC的条件渲染不同， 这里的条件渲染事实上是在子组件实现的(前面的HOC是在Wrapper实现的)， 外部的Auth函数指数提供给Children必要的props而已； 但是可不可以把UI逻辑放到Auth组件里？其实也是可以的；但是render模式一般只是提供共享的状态(比如上面的user)； 另外假设有多个子组件要共用Auth提供的状态的话， 可以考虑：

第一种方式:

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

第二种方式： 使用Context

context适合更加复杂（比如深度嵌套）的状态共享





### Props Collection 模式

#### 基础用法

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

简单来讲Props collection就是把相关的props集合在了一起， 比如上面拿到useMouse, 把位置相关的， 以及和这个位置先关的action组合在了一起

#### 结合Render Props

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

#### 多种Props集合

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

#### Props Getter 模式

不直接返回Props对象， 而是说利用闭包返回可以生成Props的函数， 这样会比较自由

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



### State Reducer模式

State Reducer 模式的核心思想是：​**​组件仍然管理自己的状态，但将状态更新的决定权通过 reducer 函数暴露给使用者​**​

#### 传统Reducer

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

#### 添加State Reducer

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

想一想为什么useReducer这种方式可以实现？ 其实就是自定义Reducer和原来的reducer拥有相同的函数签名 （state, action） => void

也正因为多个reducer共享相同的函数签名， 所以State Reducer也可以实现composer模式(类似前面组合多个HOC一样)



#### 多个reducer组合

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
  
  // 使用...
}
```

### 总结

其实上面的设计模式， 核心就是围绕

1 关注点分离， 比如Container & presention模式(比较简单，上面没有详细展开)以及Props render，  Props collections模式都是如此

2 逻辑复用， 比如HOC以及State Reducer;  前提是可复用逻辑本身和目标是存在正交性的；当然严格以上来说逻辑复用其实也是关注点分离的一种表现形式， 把重复的逻辑和主逻辑抽离出来

然后,之所以react的设计模式都需要强调关注点分类的一个重要原因在于： react本身在设计上将UI和逻辑糅合在了一起（React Hook）


