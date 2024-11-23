---
title: React常用设计模式
description: 
slug: 
date: 2024-11-23
categories:
    - web开发
tags:
    - react
weight: 1       
---
参考： [https://www.patterns.dev/react/](https://www.patterns.dev/react/)

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

> 这个模式出现在很多地方，比如Shadcn中的所有组件，也大多是基于多个更小的组件组合起来的。 不一定非得使用Flyout.xxxx来表示子组件
>

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

其他的组件也是用一同样的方式获取到value， 这个不赘述了；这里需要追加的一点的是， 隐式传递状态的方式不止上面的这种， 还可以通过React.cloneElement来实现, 比如：

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

简单来说就是直接把父组件的state作为属性附加到了FlyOut的直接子元素上，所以对于嵌套更深的组件， 这种方式就是不太行， 同时需要注意属性名冲突的问题  




## HOC
TL;DR,  HOC高阶组件就是通过函数， 抽离共用逻辑来包装已有的组件生成的组件，例如， 我们需要给一些组件添加共用属性：

```tsx
function withStyles(Component) {
  return props => {
    const style = { padding: '0.2rem', margin: '1rem' }  // 共用style
    return <Component style={style} {...props} />
  }
}

const Button = () = <button>Click me!</button>
const Text = () => <p>Hello World!</p>

const StyledButton = withStyles(Button)
const StyledText = withStyles(Text)
```

> 看起来看很简单是吧？但是这里其实隐藏了一个有意思的点， withStyle是直接作用在被包装的组件上，以Button为例， style被添加在了Button上，  但是style并没有定义在Button中的button中， 但是它依然会生效！因为React会自动声明未处理的属性， 并传递到子组件
>

对被包装Hook使用固定的style有点笨， 假设我要传入自定义的style呢？可以把style作为withStyle的第二个参数



有时候我们会使用React hooks来替代HOC， 譬如：

```tsx
// 定义一个Hook useLocalStore
import { useState, useEffect } from 'react';

interface UseLocalStoreOptions<T> {
  key: string;
  initialValue: T;
}

function useLocalStore<T>(options: UseLocalStoreOptions<T>): [T, (value: T) => void] {
  const { key, initialValue } = options;

  // 从 localStorage 中获取初始值
  const storedValue = localStorage.getItem(key);
  const initialStoredValue = storedValue ? JSON.parse(storedValue) : initialValue;

  // 使用 useState 管理状态
  const [value, setValue] = useState<T>(initialStoredValue);

  // 当 value 发生变化时，更新 localStorage
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

export default useLocalStore;

// 使用自定义useLocalStore

import React from 'react';
import useLocalStore from './useLocalStore';

const App: React.FC = () => {
  const [count, setCount] = useLocalStore({ key: 'count', initialValue: 0 });

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
};


```



首先上面这个例子可不可以使用HOC, 答案是可行的; 就是把useLocalStorageh中的和localStorage交互的部分定义到HOC， 同时把第一个参数改成Component即可（options变成第二个参数）， 只不过和之前withStyle不一样的地方在于， 我没有对子组件进行某种封装，简而言之：

+ HOC的本质就是类型为 component => component 的函数 
+ 而Hooks通常只会代理组件的一部分通用功能或者副作用（side effects）， 比如上面的从localStorage获取值的逻辑



## Render props pattern
#### 基础使用场景 - 无参render function
```tsx
// 定义一个需要带有render props的组件
const Title = ({render}) => render()

// 使用Title
export const App = () => {
  return  <Title 
            render={()=>(
              <h1>
                 This is a title
              </h1>
            )} 
            />
}
```

简单来说就是子组件将部分渲染逻辑交给父组件去执行



#### 带参数的render function
```tsx
function Input({render}) {
  const [value, setValue] = useState("");

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Temp in °C"
      />
      {render(value)}
    </>
  );
}
```





#### children
本质上render的功能就是props中chidren做的事情， 带参数的render函数本质是就是  (parms) => children

所以我们用分别改写上面的例子

```tsx
const Title = ({render}) => render()

// 等价于
const Title = ({children}) => children
```

```tsx
function Input({render}) {
  const [value, setValue] = useState("");

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Temp in °C"
      />
      {render(value)}
    </>
  );
}


// 等价于
function Input({children}) {
  const [value, setValue] = useState("");

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Temp in °C"
      />
      {children(value)}
    </>
  );
}



```

使用改写的Input组件

```tsx
function App() {
  return (
    <div className="App">
      <h1>☃️ Temperature Converter 🌞</h1>
      <Input>
        {value => (
          <>
            <Kelvin value={value} />
            <Fahrenheit value={value} />
          </>
        )}
      </Input>
    </div>
  );
}

function Kelvin({ value }) {
  return <div className="temp">{parseInt(value || 0) + 273.15}K</div>;
}

function Fahrenheit({ value }) {
  return <div className="temp">{(parseInt(value || 0) * 9) / 5 + 32}°F</div>;
}
```

其实理论上也可以用状态提升（state lifting）来实现， 但是对于第三方组件， 如果我们想要拿到它的内部状态(假设Input是第三方的组件， input中的value就是它的内部状态）， 使用render props pattern就可以轻松实现

