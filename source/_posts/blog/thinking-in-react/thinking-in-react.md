---
title: Thinking in React
author: 大大白
date: 2019-11-06 16:55:00
categories:
- WEB前端
- 最佳实践
tags:
- React哲学
---

使用 React 开发过程中的一些思考总结。持续更新...

<!-- more -->


## 变换（Transformation）
设计 React 的核心前提是认为 UI 只是把数据通过映射关系变换成另一种形式的数据。同样的输入必会有同样的输出。这恰好就是纯函数。
```js
function NameBox(name) {
  return { fontWeight: 'bold', labelContent: name };
}
'Sebastian Markbåge' ->
{ fontWeight: 'bold', labelContent: 'Sebastian Markbåge' };
```

## 抽象（Abstraction）
你不可能仅用一个函数就能实现复杂的 UI。重要的是，你需要把 UI 抽象成多个隐藏内部细节，又可复用的函数。通过在一个函数中调用另一个函数来实现复杂的 UI，这就是抽象。
```js
function FancyUserBox(user) {
  return {
    borderStyle: '1px solid blue',
    childContent: [
      'Name: ',
      NameBox(user.firstName + ' ' + user.lastName)
    ]
  };
}
{ firstName: 'Sebastian', lastName: 'Markbåge' } ->
{
  borderStyle: '1px solid blue',
  childContent: [
    'Name: ',
    { fontWeight: 'bold', labelContent: 'Sebastian Markbåge' }
  ]
};
```

## 组合（Composition）
为了真正达到重用的特性，只重用叶子然后每次都为他们创建一个新的容器是不够的。你还需要可以包含其他抽象的容器再次进行组合。我理解的“组合”就是将两个或者多个不同的抽象合并为一个。
``` js
function FancyBox(children) {
  return {
    borderStyle: '1px solid blue',
    children: children
  };
}

function UserBox(user) {
  return FancyBox([
    'Name: ',
    NameBox(user.firstName + ' ' + user.lastName)
  ]);
}
```

## 状态（State）
UI 不单单是对服务器端或业务逻辑状态的复制。实际上还有很多状态是针对具体的渲染目标。举个例子，在一个 text field 中打字。它不一定要复制到其他页面或者你的手机设备。滚动位置这个状态是一个典型的你几乎不会复制到多个渲染目标的。

我们倾向于使用不可变的数据模型。我们把可以改变 state 的函数串联起来作为原点放置在顶层。
```js
function FancyNameBox(user, likes, onClick) {
  return FancyBox([
    'Name: ', NameBox(user.firstName + ' ' + user.lastName),
    'Likes: ', LikeBox(likes),
    LikeButton(onClick)
  ]);
}

// 实现细节

var likes = 0;
function addOneMoreLike() {
  likes++;
  rerender();
}

// 初始化

FancyNameBox(
  { firstName: 'Sebastian', lastName: 'Markbåge' },
  likes,
  addOneMoreLike
);
```
注意：本例更新状态时会带来副作用（addOneMoreLike 函数中）。我实际的想法是当一个“update”传入时我们返回下一个版本的状态，但那样会比较复杂。此示例待更新

## Memoization
对于纯函数，使用相同的参数一次次调用未免太浪费资源。我们可以创建一个函数的 memorized 版本，用来追踪最后一个参数和结果。这样如果我们继续使用同样的值，就不需要反复执行它了。
```js
function memoize(fn) {
  var cachedArg;
  var cachedResult;
  return function(arg) {
    if (cachedArg === arg) {
      return cachedResult;
    }
    cachedArg = arg;
    cachedResult = fn(arg);
    return cachedResult;
  };
}

var MemoizedNameBox = memoize(NameBox);

function NameAndAgeBox(user, currentTime) {
  return FancyBox([
    'Name: ',
    MemoizedNameBox(user.firstName + ' ' + user.lastName),
    'Age in milliseconds: ',
    currentTime - user.dateOfBirth
  ]);
}
```

## 列表（Lists）
大部分 UI 都是展示列表数据中不同 item 的列表结构。这是一个天然的层级。

为了管理列表中的每一个 item 的 state ，我们可以创造一个 Map 容纳具体 item 的 state。
```js
function UserList(users, likesPerUser, updateUserLikes) {
  return users.map(user => FancyNameBox(
    user,
    likesPerUser.get(user.id),
    () => updateUserLikes(user.id, likesPerUser.get(user.id) + 1)
  ));
}

var likesPerUser = new Map();
function updateUserLikes(id, likeCount) {
  likesPerUser.set(id, likeCount);
  rerender();
}

UserList(data.users, likesPerUser, updateUserLikes);
```
注意：现在我们向 FancyNameBox 传了多个不同的参数。这打破了我们的 memoization 因为我们每次只能存储一个值。更多相关内容在下面。



## 连续性（Continuations）
不幸的是，自从 UI 中有太多的列表，明确的管理就需要大量的重复性样板代码。

我们可以通过推迟一些函数的执行，进而把一些模板移出业务逻辑。比如，使用“柯里化”（JavaScript 中的 bind，把多个参数转换成一个个接收单个参数的函数）。然后我们可以从核心的函数外面传递 state，这样就没有样板代码了。

下面这样并没有减少样板代码，但至少把它从关键业务逻辑中剥离。
```js
function FancyUserList(users) {
  return FancyBox(
    UserList.bind(null, users)
  );
}

const box = FancyUserList(data.users);
const resolvedChildren = box.children(likesPerUser, updateUserLikes);
const resolvedBox = {
  ...box,
  children: resolvedChildren
};
```

## State Map
之前我们知道可以使用组合避免重复执行相同的东西这样一种重复模式。我们可以把执行和传递 state 逻辑挪动到被复用很多的低层级的函数中去。
```js
function FancyBoxWithState(
  children,
  stateMap,
  updateState
) {
  return FancyBox(
    children.map(child => child.continuation(
      stateMap.get(child.key),
      updateState
    ))
  );
}

function UserList(users) {
  return users.map(user => {
    continuation: FancyNameBox.bind(null, user),
    key: user.id
  });
}

function FancyUserList(users) {
  return FancyBoxWithState.bind(null,
    UserList(users)
  );
}

const continuation = FancyUserList(data.users);
continuation(likesPerUser, updateUserLikes);
```

## Memoization Map
一旦我们想在一个 memoization 列表中 memoize 多个 item 就会变得很困难。因为你需要制定复杂的缓存算法来平衡调用频率和内存占有率。

还好 UI 在同一个位置会相对的稳定。相同的位置一般每次都会接受相同的参数。这样以来，使用一个集合来做 memoization 是一个非常好用的策略。

我们可以用对待 state 同样的方式，在组合的函数中传递一个 memoization 缓存。
```js
function memoize(fn) {
  return function(arg, memoizationCache) {
    if (memoizationCache.arg === arg) {
      return memoizationCache.result;
    }
    const result = fn(arg);
    memoizationCache.arg = arg;
    memoizationCache.result = result;
    return result;
  };
}

function FancyBoxWithState(
  children,
  stateMap,
  updateState,
  memoizationCache
) {
  return FancyBox(
    children.map(child => child.continuation(
      stateMap.get(child.key),
      updateState,
      memoizationCache.get(child.key)
    ))
  );
}

const MemoizedFancyNameBox = memoize(FancyNameBox);
```

## 代数效应（Algebraic Effects）
多层抽象需要共享琐碎数据时，一层层传递数据非常麻烦。如果能有一种方式可以在多层抽象中快捷地传递数据，同时又不需要牵涉到中间层级，那该有多好。React 中我们把它叫做“context”。

有时候数据依赖并不是严格按照抽象树自上而下进行。举个例子，在布局算法中，你需要在实现他们的位置之前了解子节点的大小。

现在，这个例子有一点超纲。我会使用 代数效应 这个由我发起的 ECMAScript 新特性提议。如果你对函数式编程很熟悉，它们 在避免由 monad 强制引入的仪式一样的编码。
```js
function ThemeBorderColorRequest() { }

function FancyBox(children) {
  const color = raise new ThemeBorderColorRequest();
  return {
    borderWidth: '1px',
    borderColor: color,
    children: children
  };
}

function BlueTheme(children) {
  return try {
    children();
  } catch effect ThemeBorderColorRequest -> [, continuation] {
    continuation('blue');
  }
}

function App(data) {
  return BlueTheme(
    FancyUserList.bind(null, data.users)
  );
}
```

## keys & props
推荐 clone 我的源码https://github.com/wangruolong/web-front-end/tree/master/thinking-in-react
里面有大量的注释和精心制作的运行结果，可以助你快速掌握！

### 发现问题
我们经常需要对一个数组进行循环，渲染一个列表。但是控制台经常会报警告 Warning: Each child in a list should have a unique "key" prop.有没有思考过这是为什么？
遇到这种问题的一般做法是直接把 index 赋值给 key。这样会有什么问题？
重新 render 的时候出问题了。为什么会这样？

### 分析问题
不加 key 的情况下，react 分不清哪个是哪个。
React 在循环的时候自动帮元素加上序号。
diff 算法的比较规则，优先比较 key，如果 key 不存在则比较组件名称，如果组件名称一样则 react 认为是同一个组件，不 unmount 直接重新 render。至于 props 里面的数据一不一样，React 认为这只是 props 数据变化了而已。React diff 算法可以参考这篇文章 https://www.jianshu.com/p/3ba0822018cf）
不加 key 的情况下，react 会默认加上 key，当对比到同一个 key 的组件名称不一致，就直接销毁掉了。
加 key 后，react 会根据 key 进行比较，同一个 key 如果组件名称也一样，则认为是同一个组件不 unmount，而是重新 render。

### 解决问题
为什么 input 的值会没掉？
input 有没有重新 render？但是有没有被销毁？为什么？
因为 input 的值既不是受控也不是非受控，他的值并没有被 react 维护。而值被清空了并不是说组件被销毁了，而是说重新 render 了，但是值没有被保留下来。
非受控组件。this.state.value 并没有受到 props 的控制，所以当重新 render 后，input 还是由原来的值控制。
只不过 props 改变了，这个本质上和原来是同一个组件，只不过 props 变了而已。

## hooks
为了处理在触发的时候还不存在，等到执行的时候才存在的对象。

### 发现问题
场景：一个搜索组件 Search，一个文章详情组件 Article，搜索结果返回用 i 标签包裹的关键词。
需求：搜索文章关键词，点击结果文章自动滚动到第一个关键词。

### 分析问题
实现方式：在渲染结束后，真实 Dom 上面查找 i 节点，找到 i 节点后获取它的 offset 然后设置 Article 的 scrollTop。
这就要求必须要有真实 dom 的时候才搜索，所以要写在 didUpdate，写在 didMount 也可以但是它只会触发一次所以最终还是得写在 didUpdate。
didUpdate 会被很多种情况触发，常规的做法是在 didUpdate 里面设置各种属性判断，但是这样会导致项目越来越难维护。didUpdate 里面应该尽量简单。

### 解决问题
用钩子，可以在点击的那个地方挂上一个钩子，然后在 didUpdate 那边不断循环这些钩子，一旦钩子被执行就直接销毁，这样也不会有副作用。
相当于在点击的时候 producer，然后在执行的地方 consumer，执行之后把相应的钩子销毁掉就行了。

```js
// 钩子数组，我有很多把钩子。
const hooks = [];
// 一根钩子抹点油。怎么样？我就问你这种注释看得懂吗？
function hook(subscriber) {
  if (typeof subscriber !== "function") {
    throw new Error("Invalid hook, must be a function!");
  }
  const subscriberIndex = hooks.indexOf(subscriber);
  if (subscriberIndex == -1) {
    hooks.push(subscriber);
    return () => {
      const result = hooks.splice(hooks.indexOf(subscriber), 1);
      return result[0];
    };
  } else {
    return () => {
      const result = hooks.splice(subscriberIndex, 1);
      return result[0];
    };
  }
}
// 篮子
const baskets = {};
// 出钩
function goHook(key, func) {
  baskets[key] = hook(func);
}
// 收钩。把参数带上。
// 这个...args的作用是【rest】，接收剩余参数。
function backHook(key, ...args) {
  try {
    if (baskets[key]) {
      console.log("找到钩子", baskets[key]);
      // 这个...args的作用是【spread】，把对象展开。
      baskets[key]()(...args);
      delete baskets[key];
    } else {
      console.log("没找到钩子");
    }
  } catch (e) {
    console.log(e);
  }
}

export { hooks, baskets, goHook, backHook };
```
