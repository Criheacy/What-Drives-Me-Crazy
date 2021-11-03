### 问题摘要

当使用`setState: (prevState: S) => S`的方式更新类型为列表`Array`的State时，更新函数会多执行一次

### 复现问题

1. 声明一个类型为列表的state，列表内容的类型不重要
   `const [list, setList] = useState([]);`
2. 在某个函数中调用`setList`，参数为`prevState => newState`的函数
   `setList(prev => {prev.push(123); return prev;})`
3. 执行一次第二步中的这个函数，发现`list`中多出了两个`123`

### 最短复现代码

hook-test.ts

```javascript
import {useState} from "react";

export const useListState = () => {
  const [list, setList] = useState<number[]>([]);

  const push = (value: number) => {
    setList(prevList => {
      prevList.push(value);
      return prevList;
    });
  }

  return [list, push] as const;
}
```

App.tsx


```javascript
import {useState} from "react";
import {useListState} from "./hook-test";

export const TestList = () => {
  const [count, setCount] = useState(0);
  const [list, push] = useListState();

  const handleClick = () => {
    push(count);
    setCount(count + 1);
  };

  return <div>
    <button onClick={handleClick}>add {count}</button>
    <ul>
      {list.map(item => <li>{item}</li>)}
    </ul>
  </div>;
}
```

渲染结果如下（点击三次后）：

<div><button>add 3</button><ul><li>0</li><li>0</li><li>1</li><li>1</li><li>2</li><li>2</li></ul></div>

### 尝试过的解决方法

将`setState: (prevState: S) => S`改为直接赋值的`setState: S`形式可以防止这个问题发生，但是React的`setState`具有合并机制，当短时间反复调用`setState`时React不会真正更改`state`的值，继而引发其它的问题。比如改成下面这样：

hook-test.ts

```javascript
import {useState} from "react";
import _ from "lodash";

export const useListState = () => {
  const [list, setList] = useState<number[]>([]);

  const push = (value: number) => {
    // create a new array with same elements (but different reference)
    const newList = _.cloneDeep(list);
    newList.push(value);
    setList(newList);
  }

  return [list, push] as const;
}
```

App.tsx

```javascript
import {useState} from "react";
import {useListState} from "./hook-test";

export const TestList = () => {
  const [count, setCount] = useState(0);
  const [list, push] = useListState();

  const handleClick = () => {
    // repeats 10 times
    for (let i = 0; i < 10; i++) {
      push(count);
    }
    setCount(count + 1);
  };

  return <div>
    <button onClick={handleClick}>add {count}</button>
    <ul>
      {list.map(item => <li>{item}</li>)}
    </ul>
  </div>;
}
```

在App.tsx中点击一次按钮添加十次值，但是因为`setState`的合并作用实际只添加了一次：

<div><button>add 5</button><ul><li>0</li><li>1</li><li>2</li><li>3</li><li>4</li></ul></div>

### 问题解决

> 参考：
>
> [官方Issue](https://github.com/facebook/react/issues/12856)
>
> [StackOverFlow提问](https://stackoverflow.com/a/62373252/13876457)

#### 产生原因

React中的`setState: (prevState: S) => S`还有一条隐藏的要求：这里传入的函数必须为**纯函数**（Pure Function），也就是函数要满足

1. 当输入相同时输出一定相同
2. 不产生副作用（Side Effect）

可以看出纯函数都是幂等的（Idempotent），即执行若干次与执行一次效果相同。为了确保这一点，在开发模式下且启用`Strict Mode`之后React会**故意**将所有传入函数的`setState`执行两次来帮助你确保这是一个纯函数，帮助你在开发过程中看出问题（这两次的`prevState`会传入相同的值）。

原话如下：

> It is expected that setState updaters will run twice in strict mode in development. This helps ensure the code doesn't rely on them running a single time (which wouldn't be the case if an async render was aborted and alter restarted). If your setState updaters are pure functions (as they should be) then this shouldn't affect the logic of your application.

#### 解决方案

1. （省力但是不推荐）上面说只有在开发环境才会执行两次，在运行环境下为保证性能只会运行一次，所以你可以完全忽视这个错误直接硬上线

2. 就这个样例来分析，当状态为列表时，「两次调用`setState`传入的值一样」指的其实是两次传入的列表引用对象是同一个对象，在这个对象上执行了两次`Array.prototype.push`，显然就会真的加入两个元素。可以稍加修改：

   hook-test.ts

   ```javascript
   import {useState} from "react";
   import _ from "lodash";
   
   export const useListState = () => {
     const [list, setList] = useState<number[]>([]);
   
     const push = (value: number) => {
       setList(prevList => {
         const currentList = _.cloneDeep(prevList);
           // same as: const currentList.slice() = prevList;
         currentList.push(value);
         return currentList;
       });
     }
   
     return [list, push] as const;
   }
   ```

   总之让它两遍传入相同的`value`时输出相同即可。

