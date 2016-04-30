# React - 基本理论概念

这个文档是我试图对 React 做出合理解释的心智模型。意图是通过描述依据演绎推理来的设计。

当然会有一些有争议的前提，而且这个例子本身的设计也可能有 bug 或疏忽。这只是正式确定它的最初阶段。如果你有更好的完善它的想法可以随时提交 pull request 。不透过太多库的细节而从简单到复杂的过程应该是比较合理的。

React.js 的真实实现是充满务实的方法，递增的步骤，算法优化，遗留代码，debug 工具以及其他一些可以让它真的具有高可用性的内容。另外一些东西是很短暂的，如果真的有很高的价值和优先级，随着时间流逝也会进入修改, 但是真实的实现是更加难以推导的。

我喜欢可以让自己得到训练的心智模型。

## 转变

React 核心的前提是 UI 变化是把一个数据的映射简化为一种不同的格式。同样的输入返回同样的输出。一个简单的函数。

```js
function NameBox(name) {
  return { fontWeight: 'bold', labelContent: name };
}
```

```
'Sebastian Markbåge' ->
{ fontWeight: 'bold', labelContent: 'Sebastian Markbåge' };
```

## 抽象

确实不能把一个复杂的 UI 放到一个简单的函数。但重点是 UI 可以抽象为一段可重用的代码(在使用时)不需要透露实现细节。就像函数之间的调用。

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
```

```
{ firstName: 'Sebastian', lastName: 'Markbåge' } ->
{
  borderStyle: '1px solid blue',
  childContent: [
    'Name: ',
    { fontWeight: 'bold', labelContent: 'Sebastian Markbåge' }
  ]
};
```

## 组合

为了真正达到重用的特性，只重用叶子然后每次都为他们创建一个新的容器是不够的。你还需要可以包含其他抽象的容器进行组合。我理解的“组合”就是将两个或者多个不同的抽象合并为一个。

```js
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

## 状态（state)

一个 UI 不单单是对服务器端或业务逻辑的复制。实际上还有很多状态是针对具体的某个项目的。举个例子，在一个 text field 中打字。它不一定能复用到其他页面或者你的手机设备。滚动位置是一个典型的你几乎不会复用在多个项目中的例子。

我比较偏好让我们的数据模型是不可变的。我们把可以改变 state 的函数串联起来作为原点放置在顶层。

```js
function FancyNameBox(user, likes, onClick) {
  return FancyBox([
    'Name: ', NameBox(user.firstName + ' ' + user.lastName),
    'Likes: ', LikeBox(likes),
    LikeButton(onClick)
  ]);
}

// Implementation Details

var likes = 0;
function addOneMoreLike() {
  likes++;
  rerender();
}

// Init

FancyNameBox(
  { firstName: 'Sebastian', lastName: 'Markbåge' },
  likes,
  addOneMoreLike
);
```

*NOTE: 这些例子使用副作用去更新 state。我实际的想法是当一个"update"传入时我们返回下一个版本的 state 。不使用那个也很好解释，但是在未来的版本我们会修改这些例子.*

## Memoization

当我们知道一个函数很清晰，那么反复的调用它是非常浪费资源的。所以我们为这个函数创造了一个 memorized 版本，用来追踪最后一个参数和结果。这样如果我们继续使用同样的值，就不需要反复执行它了。

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

## Lists

大多数 UI 是一种列表结构，之后会对每一个 item 产生多个不同值的列表。这是一个天然的层级。

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

*NOTE: 现在我们像 FancyNameBox 传了多个不同的参数。这打破了我们的 memoization 因为我们每次只能存储一个值。更多相关内容在下面.*

## Continuations

不幸的是，自从 UI 中有太多的列表，要想清晰的管理就需要大量的模板。

我们可以通过推迟一些函数的执行，进而把一些模板移出业务逻辑。举个例子，使用"柯里化" ([`bind`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) in JavaScript)。然后我们可以从核心的函数外面传递 state，这样我们就不依赖模板了。

并没有减少模板的数量只是至少把他们移出了重要的业务逻辑。

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

一旦我们想在一个列表 memoization 中 memoize 多个 item 就会变得很困难。你需要知道一些复杂的缓存算法来平衡频率性的内存消耗。

还好 UI 在同一个位置会适当的稳定。树种的同一位置每次都会获得同样的值。这个树对 memoization 来说是一个非常好用的策略。

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

## Algebraic Effects

如果穿过很多层级去传递每一个很小的值，这会显得有一点 PITA 。不通过调用中间件在两个组件之间传递东西是一种很好的捷径。在 React 中我们叫作 “context”。

有时候数据依赖不是很整洁的依赖组件树。举个例子，在布局算法中，你需要在实现他们的位置之前了解子节点的大小。

现在，这个例子有一点超纲。我会使用 [Algebraic Effects](http://math.andrej.com/eff/) 作为[proposed for ECMAScript](https://esdiscuss.org/topic/one-shot-delimited-continuations-with-effect-handlers)
。如果你对函数式编程很熟悉，它们在避免由 monads 带来的中间件格式的困扰。

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
    return continuation('blue');
  }
}

function App(data) {
  return BlueTheme(
    FancyUserList.bind(null, data.users)
  );
}
```
