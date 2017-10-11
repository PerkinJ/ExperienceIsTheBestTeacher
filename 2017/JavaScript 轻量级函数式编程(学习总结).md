---
title: JavaScript 轻量级函数式编程(学习总结)
category: 前端
cdn: 'header-on'
tags: [函数式编程]
---
>最近在[学习JavaScript 轻量级函数式编程](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fgetify)的教程，Kyle Simpson作者也是《You-Dont-Know-JS》的作者，感谢译者团队翻译的[《JavaScript 轻量级函数式编程》](https://juejin.im/post/599a7d8a518825242147afe8)（文中有提到），对于初学函数式编程来说，直接看英文文档还是很吃力的，毕竟很多概念不容易理解，建议看完这个系列译文之后，再回过头来看看英文教程。下面是我对学习中出现的一些概念以及重点做了一些概要，方便自己今后回顾，也方便其他小伙伴学习了解。
## 函数基础
#### 函数输入
arguments 是你输入的值（实参）， parameters 是函数中的命名变量（形参），用于接收函数的输入值。

输入计数：Arity

关于实参的小技巧
```javascript
function foo(...args) {
    console.log( args[3] );
}

var arr = [ 1, 2, 3, 4, 5 ];

foo( ...arr );                        // 4

```

关于形参的小技巧

我们都应该尽可能使用声明性的和自解释的代码。
```javascript
function foo( {x,y} = {} ) {
    console.log( x, y );
}

foo( {
    y: 3
} );     
```

#### 函数输出
因为 JS 对数组、对象和函数都使用引用和引用复制，我们可以很容易地从函数中创建输出，即使是无心的。
没有副作用的函数也有一个特殊的名称：纯函数。

#### 函数功能

> 函数是可以接受并且返回任何类型的值。一个函数如果可以接受或返回一个甚至多个函数，它被叫做高阶函数。

- 保持作用域

    当一个函数内部存在另一个函数的作用域时，对当前函数进行操作,当内部函数从外部函数引用变量，这被称作闭包。
    
- 偏函数应用和柯里化

#### 句法
> 具体来说，函数具有一个 name 的属性，用于保存函数在语法上设定名称的字符串值，例如 "helloMyNameIs" 或 "FunctionExpr"。 这个name 属性特别用于 JS 环境的控制台或开发工具。当我们在堆栈轨迹中追踪（通常来自异常）时，这个属性可以列出该函数。而如果是匿名的话，这个列表条目不给开发人员任何关于异常来源路径的线索。它没有给我们开发者提供任何帮助。

命名函数比匿名函数更可取：
- 堆栈轨迹调试
- 可靠的自我引用
- 可读性。

#### This
JavaScript 的 function 有一个 this 关键字，每个函数调用都会自动绑定。this 关键字有许多不同的方式描述，但我更喜欢说它提供了一个对象上下文来使该函数运行。

this 因为各种原因，不符合函数式编程的原则。其中一个明显的问题是隐式 this 共享。

### 管理函数的输入
#### 立即传参与稍后传参
偏函数：在函数调用现场（function call-site），将实参应用（apply） 于形参。偏函数严格来讲是一个减少函数参数个数（arity）的过程；这里的参数个数指的是希望传入的形参的数量。

柯里化（currying）技术：将一个多参数（higher-arity）函数拆解为一系列的单元链式函数。

区别：函数会明确地返回一个期望只接收下一个实参的函数，而不是那个能接收所有剩余实参的函数。

柯里化的好处是，每次函数调用传入一个实参，并生成另一个特定性更强的函数，之后我们可以在程序中获取并使用那个新函数。而偏应用则是预先指定所有将被偏应用的实参，产出一个等待接收剩下所有实参的函数。

#### 恒定参数
Certain API 禁止直接给方法传值，而要求我们传入一个函数，就算这个函数只是返回一个值。JS Promise 中的 then(..) 方法就是一个 Certain API。

#### 参数顺序的那些事儿
命名实参（named-argument）:命名实参主要的好处就是不用再纠结实参传入的顺序，因此提高了可读性。
```javascript
function partialProps(fn,presetArgsObj) {
    return function partiallyApplied(laterArgsObj){
        return fn( Object.assign( {}, presetArgsObj, laterArgsObj ) );
    };
}

function curryProps(fn,arity = 1) {
    return (function nextCurried(prevArgsObj){
        return function curried(nextArgObj = {}){
            var [key] = Object.keys( nextArgObj );
            var allArgsObj = Object.assign( {}, prevArgsObj, { [key]: nextArgObj[key] } );

            if (Object.keys( allArgsObj ).length >= arity) {
                return fn( allArgsObj );
            }
            else {
                return nextCurried( allArgsObj );
            }
        };
    })( {} );
}
```

#### 无形参风格
在函数式编程的世界中，有一种流行的代码风格，其目的是通过移除不必要的形参-实参映射来减少视觉上的干扰。这种风格的正式名称为 “隐性编程（tacit programming）”，一般则称作 “无形参（point-free）” 风格。术语 “point” 在这里指的是函数形参。

使用无形参风格的关键，是找到你代码中，有哪些地方的函数直接将其形参作为内部函数调用的实参。
```javascript
function double(x) {
    return x * 2;
}

[1,2,3,4,5].map( double );
// [2,4,6,8,10]
```

### 组合函数
#### 通用组合

我们能够像这样实现一个通用 compose(..) 实用函数：
```javascript
function compose(...fns) {
    return function composed(result){
        // 拷贝一份保存函数的数组
        var list = fns.slice();

        while (list.length > 0) {
            // 将最后一个函数从列表尾部拿出
            // 并执行它
            result = list.pop()( result );
        }

        return result;
    };
}

// ES6 箭头函数形式写法
var compose =
    (...fns) =>
        result => {
            var list = fns.slice();

            while (list.length > 0) {
                // 将最后一个函数从列表尾部拿出
                // 并执行它
                result = list.pop()( result );
            }

            return result;
        };

```

#### compose不同的实现

原始版本的 compose(..) 使用一个循环并且饥渴的（也就是，立刻）执行计算，将一个调用的结果传递到下一个调用。我们可以通过 reduce(..) （代替循环）做到同样的事。
```javascript
function compose(...fns) {
    return function composed(result){
        return fns.reverse().reduce( function reducer(result,fn){
            return fn( result );
        }, result );
    };
}

// ES6 箭头函数形式写法
var compose = (...fns) =>
    result =>
        fns.reverse().reduce(
            (result,fn) =>
                fn( result )
            , result
        );
```
这种实现的优点就是代码更简练，并且使用了常见的函数式编程结构：reduce(..)。这种实现方式的性能和原始的 for 循环版本很相近。

但是，这种实现局限处在于外层的组合函数（也就是，组合中的第一个函数）只能接收一个参数。其他大多数实现在首次调用的时候就把所有参数传进去了。如果组合中的每一个函数都是一元的，这个方案没啥大问题。但如果你需要给第一个调用传递多参数，那么你可能需要不同的实现方案。

为了修正第一次调用的单参数限制，我们可以仍使用 reduce(..) ，但加一个懒执行函数包裹器：
```javascript
function compose(...fns) {
    return fns.reverse().reduce( function reducer(fn1,fn2){
        return function composed(...args){
            return fn2( fn1( ...args ) );
        };
    } );
}

// ES6 箭头函数形式写法
var compose =
    (...fns) =>
        fns.reverse().reduce( (fn1,fn2) =>
            (...args) =>
                fn2( fn1( ...args ) )
        );
```

相较于直接计算结果并把它传入到 reduce(..) 循环中进行处理，这种实现通过在组合之前只运行 一次 reduce(..) 循环，然后将所有的函数调用运算全部延迟了 ———— 称为惰性运算。每一个简化后的局部结果都是一个包裹层级更多的函数。


我们也能够使用递归来定义 compose(..)。递归式定义的 compose(fn1,fn2, .. fnN) 看起来会是这样：
这里是我们用递归实现 compose(..) 的代码：
```javascript
function compose(...fns) {
    // 拿出最后两个参数
    var [ fn1, fn2, ...rest ] = fns.reverse();

    var composedFn = function composed(...args){
        return fn2( fn1( ...args ) );
    };

    if (rest.length == 0) return composedFn;

    return compose( ...rest.reverse(), composedFn );
}

// ES6 箭头函数形式写法
var compose =
    (...fns) => {
        // 拿出最后两个参数
        var [ fn1, fn2, ...rest ] = fns.reverse();

        var composedFn =
            (...args) =>
                fn2( fn1( ...args ) );

        if (rest.length == 0) return composedFn;

        return compose( ...rest.reverse(), composedFn );
    };
```

#### 重排序组合

从右往左顺序的标准 compose(..) 实现。这么做的好处是能够和手工组合列出参数（函数）的顺序保持一致。

不足之处就是它们排列的顺序和它们执行的顺序是相反的，这将会造成困扰。

相反的顺序，从右往左的组合，有个常见的名字：pipe(..)。

```javascript
function pipe(...fns) {
    return function piped(result){
        var list = fns.slice();

        while (list.length > 0) {
            // 从列表中取第一个函数并执行
            result = list.shift()( result );
        }

        return result;
    };
}
```

实际上，我们只需将 compose(..) 的参数反转就能定义出来一个 pipe(..)。
```javascript
var pipe = reverseArgs( compose )
```

pipe(..) 的优势在于它以函数执行的顺序排列参数，某些情况下能够减轻阅读者的疑惑。

一般来说，在使用一个完善的函数式编程库时，pipe(..) 和 compose(..) 没有明显的性能区别。

#### 抽象
抽象经常被定义为对两个或多个任务公共部分的剥离。通用部分只定义一次，从而避免重复。为了展现每个任务的特殊部分，通用部分需要被参数化。

命令式 vs 声明式的编程风格
- 命令式代码主要关心的是描述怎么做来准确完成一项任务
- 声明式代码则是描述输出应该是什么，并将具体实现交给其它部分。
```javascript
function getData() {
    return [1,2,3,4,5];
}

// 命令式
var tmp = getData();
var a = tmp[0];
var b = tmp[3];

// 声明式
var [ a ,,, b ] = getData();
```

#### 将组合当作抽象
函数组合同样也是一种声明式抽象。

组合是一个抽象的强力工具，它能够将命令式代码抽象为更可读的声明式代码。


### 减少副作用 
写出有副作用/效果的代码是很正常的， 所以我们需要谨慎和刻意地避免产生有副作用的代码。

#### 潜在的原因
- 输出和状态的变化，是最常被引用的副作用的表现。
- 随机性

    比如随机函数Math.random()

- I/O 效果

    事实上，这些来源既可以是输入也可以是输出，是因也是果。以 DOM 为例，我们更新（产生副作用的结果）一个 DOM 元素为了给用户展示文字或图片信息，但是 DOM 的当前状态是对这些操作的隐式输入（产生副作用的原因）。

- 其他的错误

#### 一次就好
如果你必须要使用副作用来改变状态，那么一种对限制潜在问题有用的操作是幂等。
- 数学中的幂等：幂等指的是在第一次调用后，如果你将该输出一次又一次地输入到操作中，其输出永远不会改变的操作。换句话说，foo(x) 将产生与 foo(foo(x))、foo(foo(foo(x))) 等相同的输出。

    比如：
    一个典型的数学例子是 Math.abs(..)（取绝对值）。Math.abs(-2) 的结果是 2，和 Math.abs(Math.abs(Math.abs(Math.abs(-2)))) 的结果相同。像Math.min(..)、Math.max(..)、Math.round(..)、Math.floor(..) 和 Math.ceil(..)这些工具函数都是幂等的。
    
- 编程中的幂等：编程中的幂等仅仅是 f(x); 的结果与 f(x); f(x) 相同而不是要求 f(x) === f(f(x))。换句话说，之后每一次调用 f(x) 的结果和第一次调用 f(x) 的结果没有任何改变。
    
    > 这种幂等性的方式经常被用于 HTTP 操作（动词），例如 GET 或 PUT。如果 HTTP REST API 正确地遵循了幂等的规范指导，那么 PUT 被定义为一个更新操作，它可以完全替换资源。同样的，客户端可以一次或多次发送 PUT 请求（使用相同的数据），而服务器无论如何都将具有相同的结果状态。


幂等与非幂等
```javascript
// 幂等的：
obj.count = 2;
a[a.length - 1] = 42;
person.name = upper( person.name );

// 非幂等的：
obj.count++;
a[a.length] = 42;
person.lastUpdated = Date.now();
```
这里的幂等性的概念是每一个幂等运算（比如 obj.count = 2）可以重复多次，而不是在第一次更新后改变程序操作。非幂等操作每次都改变状态。
```javascript
var hist = document.getElementById( "orderHistory" );

// 幂等的：
hist.innerHTML = order.historyText;

// 非幂等的：
var update = document.createTextNode( order.latestUpdate );
hist.appendChild( update );
```
这里的关键区别在于，幂等的更新替换了 DOM 元素的内容。DOM 元素的当前状态是独立的，因为它是无条件覆盖的。非幂等的操作将内容添加到元素中；隐式地，DOM 元素的当前状态是计算下一个状态的一部分。

没有副作用的函数称为**纯函数**。在编程的意义上，纯函数是一种幂等函数，因为它不可能有任何副作用。