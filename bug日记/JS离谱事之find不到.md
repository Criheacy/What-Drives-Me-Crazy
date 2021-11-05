### 有问题的代码

example.ts

用于查找并返回树结构中符合 `predicate` 判断条件的节点。调用时传入整棵树的根节点（在子树范围内查询时传入子树的根节点）。

```typescript
const findNodeInTree = (
  treeNode: TreeNodeProps,
  predicate: (node: TreeNodeProps) => unknown
): TreeNodeProps | undefined => {
  if (predicate(treeNode)) {
    return treeNode;
  }
  return treeNode.children.find(node => findNodeInTree(node, predicate));
};
```

> 如果 Typescript 版本看不懂，对应的 Javascript 版本如下：
>
> ```javascript
> const findNodeInTree = (treeNode, predicate) => {
>   if (predicate(treeNode)) {
>     return treeNode;
>   }
>   return treeNode.children.find(node => findNodeInTree(node, predicate));
> };
> ```



### 什么问题？

这块代码还真不容易看出来有什么问题，以至于排查错误的时候我就直接略过去了。

问题在于 `find` 函数字面意思虽然是“查找”元素，但它实际上只会判断 `predicate` 函数的返回值，如果返回值是真性的则返回当前判断的节点，而不是返回判断节点的返回值（即真性的那个值）。简单来说，`find(item => predicate(item))` 返回 `item` 而不是 `predicate(item)`。

比如：

```typescript
const data: {id: number, name: string}[] = [
  { id: 1, name: "one" },
  { id: 2, name: "two" },
  { id: 3, name: "three" },
  { id: 4, name: "four" },
  { id: 5, name: "five" }
];

data.find(item => item.id === 3);
// returns { id: 3, name: "three" }
// rather than 'true'
```

因此每一级的 `find` 只会返回查找到的那个子节点，而不是返回上一次 `findNodeInTree` 返回的值，这就导致最终返回的只有第一层节点。



### 解决方法

改成这样就好了：

```typescript
const findNodeInTree = (
  treeNode: TreeNodeProps,
  predicate: (node: TreeNodeProps) => unknown
): TreeNodeProps | undefined => {
  if (predicate(treeNode)) {
    return treeNode;
  }
  let result: TreeNodeProps | undefined = undefined;
  treeNode.children.forEach(node => {
    if (!result) {
      result = findNodeInTree(node, predicate);
    }
  });
  return result;
};
```

还有比较简洁的 `reduce` 写法：

```typescript
const findNodeInTree = (
  treeNode: TreeNodeProps,
  predicate: (node: TreeNodeProps) => unknown
): TreeNodeProps | undefined => {
  if (predicate(treeNode)) {
    return treeNode;
  }
  return treeNode.children.reduce((prev, node) => {
    return prev ? prev : findNodeInTree(node, predicate);
  }, undefined as TreeNodeProps | undefined);
};
```

> Javascript 写法：
>
> ```javascript
> const findNodeInTree = (treeNode, predicate)=> {
>   if (predicate(treeNode)) {
>     return treeNode;
>   }
>   return treeNode.children.reduce((prev, node) => {
>     return prev ? prev : findNodeInTree(node, predicate);
>   }, undefined);
> };
> ```

