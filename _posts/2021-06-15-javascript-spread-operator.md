---
title: "Javascript 扩展运算符"
subtitle: ""
layout: post
author: "Rucheng Tang"
header-style: text
hidden: false
tags:
  - JavaScript
  - ES6
  - 最佳实践
---

扩展运算符，`...`，最早是在 ES6（ES2015）引入的，仅用于数组，它很受欢迎，在 ES9（ES2018）为对象引入了对应的特性。本文介绍扩展运算符的实际应用。


拷贝数组或对象
------------


当我们需要修改数组或对象，但又不希望改变原有数组或对象时，就需要拷贝一份数据：


### 拷贝数组

浅拷贝（Shallow-cloning）数组时，可以使用更简短的展开语法。

```javascript
let arr = [1, 2, 3];

let arr2 = [...arr]; // like arr.slice(); or arr.map(a => a)
arr2.push(4);

console.log(arr); // [1, 2, 3]
console.log(arr2); // [1, 2, 3, 4]
```


### 拷贝对象

浅拷贝（Shallow-cloning, 不包含 prototype）对象时，可以使用更简短的展开语法，而不必使用 `Object.assign()` 方式。

```javascript
let obj = { foo: 'bar', x: 42 };

let clonedObj = { ...obj }; // like Object.assign({}, obj);
clonedObj.x = 34;

console.log(clonedObj); // {foo: "bar", x: 34}
console.log(obj); // {foo: "bar", x: 42}
```

> 如果您需要深度拷贝，推荐使用外部库（如：[Lodash](https://lodash.com/)），或自己编写一个函数来完成，下面这种方式也是一种解决方案，不过不推荐使用。
>
> ```javascript
> let obj = { foo: 'bar', x: 42, y: [1, 2] };
> 
> let clonedObj = { ...obj, y: [...obj.y] };
> clonedObj.x = 34;
> clonedObj.y.push(44);
> 
> console.log(clonedObj); // { foo: "bar", x: 34, y: Array [1, 2, 44] }
> console.log(obj); // { foo: "bar", x: 42, y: Array [1, 2] }
> ```

合并数组或对象
------------

### 合并数组

```javascript
let a = [1, 2, 3];
let b = [4, 5, 6];

let mergeab = [...a, ...b]; // like a.concat(b); or a.push.apply(a, b);

console.log(mergeab); // [1, 2, 3, 4, 5, 6]
```

### 合并对象

```javascript
let obj1 = { foo: 'bar', x: 42 };
let obj2 = { foo: 'baz', y: 13 };

let mergedObj = {...obj1, ...obj2}; // like Object.assign({}, obj1, obj2);

console.log(mergedObj); // {foo: "baz", x: 42, y: 13}
```


向数组或对象中添加元素
-------------------


### 向数组中添加元素

```javascript
let a = [1, 2, 3];
let b = [...a, 4, 5, 6];

console.log(b); // [1, 2, 3, 4, 5, 6]
```


### 向对象中添加元素

```javascript
let person = {
    name: 'songzhenzhong',
    age: 8
}
let person2 = {
    ...person,
    birthday: '1941',
    death_date: '1949-09-06'
}

console.log(person2); // {name: "songzhenzhong", age: 8, birthday: "1941", death_date: "1949-09-06"}
```

正如您所看到的，我们可以直接在对象字面量内部声明和初始化属性，而不是在外面。


唯一数组
-------

如果我们想从数组中筛选出重复的元素，那么最简单的解决方案是什么？

`Set` 对象仅存储唯一的元素，并且可以用数组填充。它也是可迭代的，因此我们可以将其展开到新的数组中，并且得到的数组中的值是唯一的。

```javascript
let a = [1, 2, 2, 3, 4, 4, 5];
let b = [...new Set(a)]; // like a.filter((currentValue, index, arr) => arr.indexOf(currentValue) === index);

console.log(b); // [1, 2, 3, 4, 5]
```


将类数组（array-like）结构转换为数组
--------------------------------

类数组结构与数组非常相似，它们通常具有编号元素（key）和长度属性。然而，它们有一个重要的区别，类数组结构没有任何数组函数。

使用与拷贝数组相同的语法，我们可以使用扩展运算符将类数组结构转换为数组，作为使用 `Array.from()` 的替代方法。看一个将 nodeList 转换为数组的示例：

```javascript
let nodeList = document.getElementsByClassName("moon");
let array = [...nodeList]; // like Array.prototype.slice(nodeList);
  
console.log(nodeList); //Result: HTMLCollection [ div.moon, div.moon ]
console.log(array); //Result: Array [ div.moon, div.moon ]
```

使用这种技术，我们可以将任何类似数组的结构转换为数组，从而可以访问所有数组函数。

再例如将遍历器转为数组：

```javascript
let string = 'test1test2test3';
let regex = /t(e)(st(\d?))/g;

let result = [...string.matchAll(regex)];
```


将参数作为数组进行传递
-------------------

```javascript
let numbers = [1, 4, 5, 6, 9, 2, 3, 4, 5, 6];
let max = Math.max(...numbers); // like Math.max.apply(null, numbers);

console.log(max); // 9
```


将字符串拆分为字符
---------------

```javascript
let text = 'The best time';

let textArray = [...text]; // like text.split('');

console.log(textArray); // ["T", "h", "e", " ", "b", "e", "s", "t", " ", "t", "i", "m", "e"]
```


剩余参数
-------

函数的最后一个参数可以以 `...` 为前缀，这将导致所有剩余的（用户提供的）参数放置在标准 javascript 数组中。只有最后一个参数可以是剩余参数。

```javascript
function myFun(a,  b, ...manyMoreArgs) {
  console.log("a", a)
  console.log("b", b)
  console.log("manyMoreArgs", manyMoreArgs)
}

myFun("one", "two", "three", "four", "five", "six")

// Console Output:
// a, one
// b, two
// manyMoreArgs, ["three", "four", "five", "six"]
```

解构时也可以使用剩余参数，它允许我们获取对象中剩余的属性，并存储它们。

```javascript
let pokemon = {
  id: 1,
  name: 'Squirtle',
  type: 'Water'
};

let { id, ...rest } = pokemon;
console.log(rest); // { name: 'Squirtle', type: 'Water' }
```

数组同理，数组切片应用：

```javascript
let a = [1, 2, 3];
[one, ...b] = a; // like b = a.slice(1);

console.log(b); // [2, 3]
```

剩余参数可以被解构，这意味着它们的数据可以被解包到不同的变量中。

```javascript
function f(...[a, b, c]) {
  return a + b + c;
}

f(1)          // NaN (b and c are undefined)
f(1, 2, 3)    // 6
f(1, 2, 3, 4) // 6 (the fourth parameter is not destructured)
```

### 将 `arguments` 对象转为数组

剩余参数通常用于将一组参数转换为数组。

**剩余参数和 `arguments` 对象的区别**

- 剩余参数只包含那些没有对应形参的实参，而 `arguments` 对象包含了传给函数的所有实参。
- `arguments` 对象不是一个真正的数组，而剩余参数是真正的 `Array` 实例，也就是说你能够在它上面直接使用所有的数组方法，比如 `sort`，`map`，`forEach` 或 `pop`。
- `arguments` 对象还有一些附加的属性 （如 `callee` 属性）。

```javascript
function f(...args) {
  let normalArray = args
  let first = normalArray.shift() // OK, gives the first argument
}
```

在 `arguments` 对象上使用 `Array` 方法

```javascript
function f(a, b) {
  let normalArray = Array.prototype.slice.call(arguments)
  // -- or --
  let normalArray = [].slice.call(arguments)
  // -- or --
  let normalArray = Array.from(arguments)

  let first = normalArray.shift()  // OK, gives the first argument
  let first = arguments.shift()    // ERROR (arguments is not a normal array)
}
```


参考文章
-------

1. [Spread syntax (...)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
2. [Rest parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters)
3. [Understanding the JavaScript Spread Operator — From Beginner to Expert](https://betterprogramming.pub/understanding-the-javascript-spread-operator-from-beginner-to-expert-8f1c110c64db)
4. [Understanding the JavaScript Spread Operator — Advanced Uses](https://betterprogramming.pub/understanding-the-javascript-spread-operator-from-beginner-to-expert-part-2-1ec1808d015e)
5. [Javascript 中的 ...（展开运算符）](https://blog.csdn.net/qq_43258252/article/details/103007068)
