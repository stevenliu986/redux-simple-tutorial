# Redux 进阶教程
> 原文链接：https://github.com/kenberkeley/redux-simple-tutorial/blob/master/redux-advanced-tutorial.md

> ### 写在前面  
> 相信您已经看过 [Redux 简明教程][simple-tutorial]，本教程是简明教程的实战化版本，伴随源码分析  
> Redux 用的是 ES6 编写，看到有疑惑的地方的，可以复制粘贴到[这里][babel-repl]在线编译查看

## &sect; Redux API 总览
在 Redux 的[源码目录][redux-src] `src/`，我们可以看到如下文件结构：

```
├── utils/
│     ├── warning.js # 打酱油的，负责在控制台显示警告信息
├── applyMiddleware.js
├── bindActionCreators.js
├── combineReducers.js
├── compose.js
├── createStore.js
├── index.js # 入口文件
```

除去打酱油的 `utils/warning.js` 以及入口文件 `index.js`，剩下那 5 个就是 Redux 的 API

## &sect; Redux API · [compose(...functions)][compose]
> 先说这个 API 的原因是它没有依赖，是一个纯函数

### ⊙ 源码分析

```
/**
 * 看起来逼格很高的样子，实际上作用就是：
 * compose(f, g, h)(...arg) => f(g(h(...args)))
 *
 * 值得注意的是，它用到了 reduceRight，因此执行顺序是从右到左
 *
 * @param  {多个函数，用逗号隔开}
 * @return {函数}
 */

export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  const last = funcs[funcs.length - 1]
  const rest = funcs.slice(0, -1)
  return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
}
```

这里的关键点在于，`reduceRight` 可传入初始值。由于 `reduce / reduceRight` 仅仅是方向的不同，因此下面用 `reduce` 说明即可：

```
var arr = [1, 2, 3, 4, 5]

var re1 = arr.reduce(function(total, i) {
  return total + i
})
console.log(re1) // 15

var re2 = arr.reduce(function(total, i) {
  return total + i
}, 100) // 传入一个初始值
console.log(re2) // 115
```

## &sect; Redux API · [createStore(reducer, [initialState])][createStore]
### ⊙ 源码分析

```
import isPlainObject from 'lodash/isPlainObject'
import $$observable from 'symbol-observable'

/**
 * 这是 Redux 的私有 action
 * 长得太丑了，你不要鸟就行了
 */
export var ActionTypes = {
  INIT: '@@redux/INIT'
}

/**
 * @param  {函数}  reducer 不多解释了
 * @param  {对象}  preloadedState 主要用于前后端同构时的数据同步
 * @param  {函数}  enhancer 很牛逼，可以实现中间件、时间旅行，持久化等
 * （目前 Redux 中仅提供 appleMiddleware 这个 store enhancer）
 * @return {Store} 全局唯一的 store 实例
 */
export default function createStore(reducer, preloadedState, enhancer) {
  // 省略一大坨类型判断
  
  var currentReducer = reducer
  var currentState = preloadedState // 这就是整个应用的 state
  var currentListeners = [] // 用于存储订阅的回调函数，dispatch 后逐个执行
  var nextListeners = currentListeners // 【悬念1：为什么需要两个 存放回调函数 的变量？】
  var isDispatching = false

  /**
   * 【悬念1·解疑】
   * 试想，dispatch 后，回调函数正在乖乖地被逐个执行（for 循环进行时）
   * 假设回调函数队列原本是这样的 [a, b, c, d]
   *
   * 现在 for 循环执行到第 3 步，亦即 a、b 已经被执行，准备执行 c
   * 但在这电光火石的瞬间，a 被取消订阅！！！
   *
   * 那么此时回调函数队列就变成了 [b, c, d]
   * 那么第 3 步就对应换成了 d！！！
   * c 被跳过了！！！这就是躺枪
   * 
   * 作为一个回调函数，最大的耻辱就是得不到执行
   * 因此为了避免这个问题，下面这个函数就把
   * currentListeners 复制给 nextListeners
   *
   * 这样的话，dispatch 后，在逐个执行回调函数的过程中
   * 如果有新增订阅或取消订阅，都在 nextListeners 中操作
   * 让 currentListeners 中的回调函数得以完整地执行
   *
   * 既然新增是在 nextListeners 中 push
   * 因此新的回调函数不会在本次 currentListeners 的循环体中被触发
   *
   * （上述事件发生的几率虽然很低，但还是严谨点比较好）
   */
  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  /**
   * 返回 state
   */
  function getState() {
    return currentState
  }

  /**
   * 负责注册回调函数的老司机
   * 
   * 这里需要注意的就是，回调函数中如果需要获取 state
   * 那每次获取都请使用 getState() 而不是开头用一个变量缓存住它
   * 因为回调函数执行期间，有可能有连续几个 dispatch 让 state 物是人非
   * 而且别忘了，dispatch 之后，整个 state 是被完全替换掉的
   * 你缓存的 state 指向的是老掉牙的 state 了！！！
   *
   * @param  {函数} 想要订阅的回调函数
   * @return {函数} 取消订阅的函数
   */
  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected listener to be a function.')
    }

    var isSubscribed = true

    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    // 返回一个取消订阅的函数
    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      var index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

  /**
   * 改变应用状态 state 的不二法门：dispatch 一个 action
   * 内部的实现是：往 reducer 中传入 currentState 以及 action
   * 用其返回值替换 currentState，最后就是逐个触发回调函数
   *
   * 如果 dispatch 的不是一个对象类型的 action（同步的），而是 Promise / thunk（异步的）
   * 则需引入 redux-thunk 等中间件来反转控制权【悬念2：什么是反转控制权？】
   * 
   * @param & @return {对象} action
   */
  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
        'Use custom middleware for async actions.'
      )
    }

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
        'Have you misspelled a constant?'
      )
    }

    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    try {
      isDispatching = true
      // 关键点：currentState 与 action 会流通到所有的 reducer
      // 所有 reducer 的返回值整合后，替换掉当前的 currentState
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    // 令 currentListeners 等于 nextListeners
    // 这是上面 ensureCanMutateNextListeners 函数的判定条件
    var listeners = currentListeners = nextListeners

    // 逐个触发回调函数。这里不保存数组长度是明智的，原因见【悬念1·解疑】
    for (var i = 0; i < listeners.length; i++) {
      listeners[i]()
    }

    return action // 作者为了方便链式调用，于是返回 action（下面会提到的）
  }

  /**
   * 替换当前 reducer 的老司机
   * 主要用于代码分离按需加载、热替换等情况
   *
   * @param {函数} nextReducer
   */
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer
    dispatch({ type: ActionTypes.INIT })
  }

  /**
   * 这是留给 可观察/响应式库 的接口（详情 https://github.com/zenparsing/es-observable）
   * 如果您了解 RxJS 等响应式编程库，那可能会用到这个接口，否则请略过
   * @return {observable}
   */
  function observable() {
    var outerSubscribe = subscribe
    return {
      /**
       * @param  {observer}
       * @return {subscription}
       */
      subscribe(observer) {
        if (typeof observer !== 'object') {
          throw new TypeError('Expected the observer to be an object.')
        }

        function observeState() {
          if (observer.next) {
            observer.next(getState())
          }
        }

        observeState()
        var unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
  }

  // store 生成后，这个 INIT action 将会被 dispatch，得到应用的初始状态
  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}

```

【悬念2：什么是反转控制权？ · 解疑】  
在同步场景下，`dispatch(action)` 的这个 `action` 中的数据是同步获取的，并没有控制权的切换问题  
但异步场景下，则需要将 `dispatch` 传入到回调函数，待异步操作完成后，回调函数自行调用 `dispatch(action)`  

说白了：在异步 Action Creator 中调用 `dispatch` 就相当于反转控制权  
您完全可以自己实现，也可以借助 [redux-thunk][redux-thunk] / [redux-promise][redux-promise] 实现

> 拓展阅读：阮老师的 [Thunk 函数的含义与用法][ryf-thunk]

## &sect; Redux API · [combineReducers(reducers)][combineReducers]
### ⊙ 应用场景

简明教程中的 `code-7` 如下：

```js
/** 本代码块记为 code-7 **/
var initState = {
  counter: 0,
  todos: []
}

function reducer(state, action) {
  if (!state) state = initState
  
  switch (action.type) {
    case 'ADD_TODO':
      var nextState = _.deepClone(state) // 用到了 lodash 的深克隆
      nextState.todos.push(action.payload) 
      return nextState

    default:
      return state
  }
}
```

上面的 `reducer` 仅仅是实现了 “新增待办事项” 的 `state` 的处理  
我们还有计数器的功能，下面我们继续增加计数器 “增加 1” 的功能：

```js
/** 本代码块记为 code-8 **/
var initState = { counter: 0, todos: [] }

function reducer(state, action) {
  if (!state) return initState // 若是初始化可立即返回应用初始状态
  var nextState = _.deepClone(state) // 否则二话不说先克隆
  
  switch (action.type) {
    case 'ADD_TODO': // 新增待办事项
      nextState.todos.push(action.payload) 
      break   
    case 'INCREMENT': // 计数器加 1
      nextState.counter = nextState.counter + 1
      break
  }
  return nextState
}
```

如果说还有其他的动作，都需要在 `code-8` 这个 `reducer` 中继续堆砌处理逻辑  
但我们知道，计数器 与 待办事项 属于两个不同的模块，不应该都堆在一起写  
如果之后又要引入新的模块（例如留言板），该 `reducer` 会越来越臃肿  
此时就是 `combineReducers` 大显身手的时刻：

```
目录结构如下
reducers/
   ├── index.js
   ├── counterReducer.js
   ├── todosReducer.js
```

```
/** 本代码块记为 code-9 **/
/* reducers/index.js */
import { combineReducers } from 'redux'
import counterReducer from './counterReducer'
import todosReducer from './todosReducer'

const rootReducer = combineReducers({
  counter: counterReducer,
  todos: todosReducer
})

export default rootReducer
```

```
/* reducers/counterReducer.js */
export default function counterReducer(counter = 0, action) {
  switch (action.type) {
    case 'INCREMENT':
      return counter + 1 // counter 是值传递，因此可以直接返回一个值
    default:
      return counter
  }
}
```

```
/* reducers/todosReducers */
export default function todosReducer(todos = [], action) {
  switch (action.type) {
    case 'ADD_TODO':
      return [ ...todos, action.payload ]
    default:
      return todos
  }
}
```

`code-8 reducer` 与 `code-9 rootReducer` 的功能是一样的，但后者的各个子 `reducer` 仅维护对应的那部分 `state`  
其可操作性、可维护性、可扩展性大大增强

> Flux 中是根据不同的功能拆分出多个 `store` 分而治之  
> 而 Redux 只允许应用中有唯一的 `store`，通过拆分出多个 `reducer` 分别管理对应的 `state`

***

下面继续来深入使用 `combineReducers`。一直以来我们的应用状态都是只有两层，如下所示：

```
state
  ├── counter: 0
  ├── todos: []
```

如果说现在又有一个需求：在待办事项模块中，存储用户每次操作（增删改）的时间，那么此时应用初始状态树应为：

```
state
  ├── counter: 0
  ├── todo
        ├── optTime: []
        ├── todoList: [] # 这就是原来的 todos！
```

那么对应的 `reducer` 就是：

```
目录结构如下
reducers/ <-------------------- combineReducers (生成 rootReducer)
   ├── index.js
   ├── counterReducer.js
   ├── todoReducers/ <--------- combineReducers
           ├── index.js
           ├── optTimeReducer.js
           ├── todoListReducer.js
```

```
/* reducers/index.js */
import { combineReducers } from 'redux'
import counterReducer from './counterReducer'
import todoReducers from './todoReducers/'

const rootReducer = combineReducers({
  counter: counterReducer,
  todo: todoReducers
})

export default rootReducer
```

```
/* reducers/todoReducers/index.js */
import { combineReducers } from 'redux'
import optTimeReducer from './optTimeReducer'
import todoListReducer from './todoListReducer'

const todoReducers = combineReducers({
  optTime: optTimeReducer,
  todoList: todoListReducer
})

export default todoReducers
```

```
/* reducers/todosReducers/optTimeReducer.js */
export default function optTimeReducer(optTime = [], action) {
  // 咦？这里怎么没有 switch-case 分支？谁说 reducer 就一定包含 switch-case 分支的？
  return action.type.includes('TODO') ? [ ...optTime, new Date() ] : optTime
}
```

```
/* reducers/todosReducers/todoListReducer.js */
export default function todoListReducer(todoList = [], action) {
  switch (action.type) {
    case 'ADD_TODO':
      return [ ...todoList, action.payload ]
    default:
      return todoList
  }
}
```
  
无论您的应用状态树有多么的复杂，都可以通过层层分支，分而治之地管理对应部分的 `state`：

```
                                 counterReducer(counter, action) -------------------- counter
                              ↗                                                              ↘
rootReducer(state, action) —→       ↗ optTimeReducer(optTime, action) ------ optTime ↘         nextState
                              ↘ —→                                                      todo  ↗
                                    ↘ todoListReducer(todoList,action) ----- todoList ↗
```

> 无论是 `dispatch` 哪个 `action`，都会流通**所有的** `reducer`  
> 表面上看来，这样子很浪费性能，但 JavaScript 对于这种纯函数的调用是很高效率的，因此请尽管放心  
> 这也是为何 `reducer` 必须返回其对应的 `state` 的原因。否则整合状态树时，该 `reducer` 对应的键名就是 `undefined`

### ⊙ 源码分析
> 仅截取关键部分，毕竟有很大一部分都是类型检测警告

```
function combineReducers(reducers) {
  var reducerKeys = Object.keys(reducers)
  var finalReducers = {}
  
  for (var i = 0; i < reducerKeys.length; i++) {
    var key = reducerKeys[i]
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }

  var finalReducerKeys = Object.keys(finalReducers)

  // 返回一个 reducer
  return function combination(state = {}, action) {
    var hasChanged = false
    var nextState = {}
    for (var i = 0; i < finalReducerKeys.length; i++) {
      var key = finalReducerKeys[i]
      var reducer = finalReducers[key]
      var previousStateForKey = state[key]
      var nextStateForKey = reducer(previousStateForKey, action) // 传入各子 reducer 中获取 nextState
      nextState[key] = nextStateForKey // 关键点：将对应的 state 挂载到对应的键名
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    return hasChanged ? nextState : state
  }
}

```

> 在此我的注释很少，因为代码写得实在是太过明了了，注释反而影响阅读  
> 作者 Dan 用了大量的 `for` 循环，的确有点不够优雅

## &sect; Redux API · [bindActionCreators(actionCreators, dispatch)][bindActionCreators]
> 这个 API 有点鸡肋，它无非就是做了这件事情：`dispatch(ActionCreator(XXX))`

### ⊙ 源码分析

```
// 为 Action Creator 加装上自动 dispatch 技能
function bindActionCreator(actionCreator, dispatch) {
  return (...args) => dispatch(actionCreator(...args))
}

export default function bindActionCreators(actionCreators, dispatch) {
  // 省去一大坨类型判断
  var keys = Object.keys(actionCreators)
  var boundActionCreators = {}
  for (var i = 0; i < keys.length; i++) {
    var key = keys[i]
    var actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      // 逐个装上自动 dispatch 技能
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}
```

### ⊙ 应用场景
简明教程中的 `code-5` 如下：

```html
<--! 本代码块记为 code-5 -->
<input id="todoInput" type="text" />
<button id="btn">提交</button>

<script>
$('#btn').on('click', function() {
  var content = $('#todoInput').val() // 获取输入框的值
  var action = addTodo(content) // 执行 Action Creator 获得 action
  store.dispatch(action) // 手动显式 dispatch 一个 action
})
</script>
```

我们看到，调用 `addTodo` 这个 Action Creator 后得到一个 `action`，之后又要 `dispatch(action)`  
如果是只有一个两个 Action Creator 还是可以接受，但如果有很多个那就显得有点重复了（其实我觉得不重复哈哈哈）  
这个时候我们就可以利用 `bindActionCreators` 实现自动 `dispatch`：

```html
<input id="todoInput" type="text" />
<button id="btn">提交</button>

<script>
// Redux 全局引入，同时 store 是全局变量
var actionsCreators = Redux.bindActionCreators(
  { addTodo: addTodo },
  store.dispatch // 传入 dispatch 函数
)

$('#btn').on('click', function() {
  var content = $('#todoInput').val()
  actionCreators.addTodo(content) // 好吧，我承认的确是更复杂了哈哈哈
})
</script>
```

> 综上，这个 API 没啥卵用，尤其是异步场景下，基本用不上

## &sect; Redux API · [applyMiddleware(...middlewares)][applyMiddleware]
> Redux 中文文档 [高级 · Middleware][redux-middleware] 有提到中间件的演化由来

首先要理解何谓 `Middleware`，何谓 `Enhancer`

### ⊙ Middleware
说白了，中间件其实就是**统一处理**与业务无关的东西，并且不更改原有的 `store` API  
诸如日志记录、引入 thunk 处理异步 Action 等都属于中间件。下面是一个简单的打印动作前后 `state` 的中间件：

```
// 装逼写法
const printStateMiddleware = ({ getState }) => next => action => {
  console.log('state before dispatch', getState())
  
  let returnValue = next(action)

  console.log('state after dispatch', getState())

  return returnValue
}
```

```
// 降低逼格写法
function printStateMiddleware(middlewareAPI) { // 记为 锚点-1，中间件可用的 API
  return function (dispatch) {                 // 记为 锚点-2，传入原 store.dispatch 的引用
    return function (action) {
      console.log('state before dispatch', middlewareAPI.getState())
  
      var returnValue = dispatch(action) // dispatch 的返回值其实还是 action
  
      console.log('state after dispatch', middlewareAPI.getState())

      return returnValue // 作为参数 action，继续传给下一个中间件
    }
  }
}
```

### ⊙ Store Enhancer
说白了，Store 增强器就是对 `store` 的功能进行增强。例如，Redux 的 API `applyMiddleware` 实际上就是一个 Store 增强器  
话不多说，直接上源码：

```
import compose from './compose' // 还记得吗？这货的作用就是 compose(f, g, h)(action) => f(g(h(action)))

export default function applyMiddleware(...middlewares) {

  /* 传入 Redux 的 API：createStore */
  return function(createStore) {
  
    /* 返回一个函数，函数签名跟 createStore 一模一样，亦即返回的是一个增强版的 createStore */
    return function(reducer, preloadedState, enhancer) {
    
      // 生成一个常规的 store，其包含 getState / dispatch / subscribe / replaceReducer 四个 API
      var store = createStore(reducer, preloadedState, enhancer)
      
      var dispatch = store.dispatch // 指向原 dispatch
      var chain = [] // 存储中间件的数组
  
      // 提供给中间件的 API（都是 store 的 API）
      var middlewareAPI = {
        getState: store.getState,
        dispatch: (action) => dispatch(action)
      }
      
      // 给中间件“装上” API，见上述【降低逼格写法】的 锚点-1 
      chain = middlewares.map(middleware => middleware(middlewareAPI))
      
      // 串联各个中间件，为各个中间件传入原 store.dispatch，见上述【降低逼格写法】的 锚点-2
      dispatch = compose(...chain)(store.dispatch)
  
      return {
        ...store, // store 的 API 中保留 getState / subsribe / replaceReducer
        dispatch  // 新 dispatch 覆盖原 dispatch，之后调用 dispatch 就会触发 chain 内的中间件链式串联执行
      }
    }
  }
}

```

上面最终返回的还是 `store` 的那四个 API，但其中的 `dispatch` 函数的功能被增强了，这就是所谓 Store Enhancer 的作用

### ⊙ 综合应用 ( [在线演示][jsbin] )
```
<!DOCTYPE html>
<html>
<head>
  <script src="//cdn.bootcss.com/redux/3.5.2/redux.min.js"></script>
</head>
<body>
<script>
/** Action Creators */
function inc() {
  return { type: 'INCREMENT' };
}
function dec() {
  return { type: 'DECREMENT' };
}

function reducer(state, action) {
  state = state || { counter: 0 };

  switch (action.type) {
    case 'INCREMENT':
      return { counter: state.counter + 1 };
    case 'DECREMENT':
      return { counter: state.counter - 1 };
    default:
      return state;
  }
}

function printStateMiddleware(middlewareAPI) {
  return function (dispatch) {
    return function (action) {
      console.log('dispatch 前：', middlewareAPI.getState());
      var returnValue = dispatch(action);
      console.log('dispatch 后：', middlewareAPI.getState());
      return returnValue;
    };
  };
}
  
var enhancedCreateStore = Redux.applyMiddleware(printStateMiddleware)(Redux.createStore);
var store = enhancedCreateStore(reducer);

store.dispatch(inc());
store.dispatch(inc());
store.dispatch(dec());
</script>
</body>
</html>
```

***

## [&sect; 最后：React + Redux + React Router 实例][react-demo]

[simple-tutorial]: https://github.com/kenberkeley/react-simple-tutorial
[babel-repl]: http://babeljs.io/repl/
[redux-src]: https://github.com/reactjs/redux/tree/master/src
[deep-in-redux]: http://zhenhua-lee.github.io/react/redux.html
[redux-thunk]: https://github.com/gaearon/redux-thunk
[redux-promise]: https://github.com/acdlite/redux-promise
[ryf-thunk]: http://www.ruanyifeng.com/blog/2015/05/thunk.html
[compose]: http://cn.redux.js.org/docs/api/compose.html
[createStore]: http://cn.redux.js.org/docs/api/createStore.html
[combineReducers]: http://cn.redux.js.org/docs/api/combineReducers.html
[bindActionCreators]: http://cn.redux.js.org/docs/api/bindActionCreators.html
[applyMiddleware]: http://cn.redux.js.org/docs/api/applyMiddleware.html
[redux-middleware]: http://cn.redux.js.org/docs/advanced/Middleware.html
[jsbin]: http://jsbin.com/luhira/edit?html,console
[react-demo]: https://github.com/kenberkeley/react-demo