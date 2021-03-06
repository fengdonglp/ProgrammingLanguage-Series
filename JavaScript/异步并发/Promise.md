[![返回目录](https://parg.co/USw)](https://parg.co/bxN)

# Promise/A 内部原理与常见接口实现

```js
try {
  let arrayLike = {
    0: Promise.resolve('233'),
    length: 1
  };
  Promise.all(arrayLike);
} catch (e) {
  console.log('error');
}

var promises = [
  new Promise(function(resolve) {
    setTimeout(function() {
      resolve(1);
    }, 90);
  }),
  new Promise(function(resolve) {
    setTimeout(function() {
      resolve(2);
    }, 10);
  }),
  new Promise(function(resolve) {
    setTimeout(function() {
      resolve(3);
    }, 50);
  })
];

Promise.all(promises).then(console.log);
```

# 异步设计模式

在日常的项目开发中我们经常会需要处理异步调用，本部分我们即讨论如何以回调、Promise 以及  async/await 来实现常见的异步模式。

## 失败重试

```js
function requestWithRetry(url, retryCount) {
  if (retryCount) {
    return new Promise((resolve, reject) => {
      const timeout = Math.pow(2, retryCount);

      setTimeout(() => {
        console.log('Waiting', timeout, 'ms');
        _requestWithRetry(url, retryCount)
          .then(resolve)
          .catch(reject);
      }, timeout);
    });
  } else {
    return _requestWithRetry(url, 0);
  }
}

function _requestWithRetry(url, retryCount) {
  return request(url, retryCount).catch(err => {
    if (err.statusCode && err.statusCode >= 500) {
      console.log('Retrying', err.message, retryCount);
      return requestWithRetry(url, ++retryCount);
    }
    throw err;
  });
}

requestWithRetry('http://localhost:3000')
  .then(res => {
    console.log(res);
  })
  .catch(err => {
    console.error(err);
  });
```

```js
function wait(timeout) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve();
    }, timeout);
  });
}

async function requestWithRetry(url) {
  const MAX_RETRIES = 10;
  for (let i = 0; i <= MAX_RETRIES; i++) {
    try {
      return await request(url);
    } catch (err) {
      const timeout = Math.pow(2, i);
      console.log('Waiting', timeout, 'ms');
      await wait(timeout);
      console.log('Retrying', err.message, i);
    }
  }
}
```

## 数组转换

```js
function asyncThing(value) {
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(value), 100);
  });
}

async function mapArray() {
  return [1, 2, 3, 4].map(async value => {
    const v = await asyncThing(value);
    return v * 2;
  });
}

async function filterArray() {
  return [1, 2, 3, 4].filter(async value => {
    const v = await asyncThing(value);
    return v % 2 === 0;
  });
}

async function reduceArray() {
  return [1, 2, 3, 4].reduce(async (acc, value) => {
    return (await acc) + (await asyncThing(value));
  }, Promise.resolve(0));
}

// [ Promise { <pending> }, Promise { <pending> }, Promise { <pending> }, Promise { <pending> } ]
```

```javascript
filterArray().then(v => {
  console.log(v);
});

// [ 1, 2, 3, 4 ]
```

```js
typeof new Promise((resolve, reject) => {}) === 'object'; // true
```

Promise 本质上只是普通的 JavaScript 对象，包含了允许你执行某些异步代码的方法。

```js
const fetch = function(url) {
  return new Promise((resolve, reject) => {
    request((error, apiResponse) => {
      if (error) {
        reject(error);
      }

      resolve(apiResponse);
    });
  });
};
```

```js
class SimplePromise {
  constructor(executionFunction) {
    this.promiseChain = [];
    this.handleError = () => {};

    this.onResolve = this.onResolve.bind(this);
    this.onReject = this.onReject.bind(this);

    // 这里就可以发现传入的带回调的函数时立即执行的
    executionFunction(this.onResolve, this.onReject);
  }

  then(onResolve) {
    this.promiseChain.push(onResolve);

    return this;
  }

  catch(handleError) {
    this.handleError = handleError;

    return this;
  }

  onResolve(value) {
    let storedValue = value;

    try {
      this.promiseChain.forEach(nextFunction => {
        storedValue = nextFunction(storedValue);
      });
    } catch (error) {
      this.promiseChain = [];

      this.onReject(error);
    }
  }

  onReject(error) {
    this.handleError(error);
  }
}
```

当我们使用 `new Promise((resolve, reject) => {/* ... */})` 这样的形式去创建 Promise 对象时，传入的 executionFunction 函数的两个参数 resolve 与 reject， 实际上映射到了 SimplePromise 类的 onResolve 与 onReject 方法。而构造器同样会创建内置的 promiseChain 数组，该数组用于记录通过 then 设置的异步传入值；而 handleError 则用于响应 onReject 回调。在原生的 Promise 实现中，then 与 catch 都返回的是新的 Promise 对象，在 SimplePromise 的实现中我们则忽略了这一步。另外，原生的 Promise 实现中允许添加多个 catch 回调，并且不需要跟随在 then 后面。

Promise.all

Promise.finally

connect

combineReducer

```js
polyfill redux connect()
         redux combineReducer()

function connect(mapStateToProps, mapDispatchToProps){

  // 构造好的封装组件
  class HOCComponent extends Component{


    // 利用 Context 进行状态传递, Todo
    getChildContext(){

    }

    handleProps(){

      let propsFromState;

      if(typeof mapStateToProps === "function"){
       // 计算映射之后的 Props 值
       propsFromState = mapStateToProps(state);

      }else{

        propsFromState = mapStateToProps;
      }

      return {...propsFromState, ...mapDispatchToProps}

    }

    render(){

      // 将新的 Props 映射入组件
      return React.createElement(HOCComponent.WrappedComponent, this.handleProps(mapStateToProps, mapDispatchToProps))

    }

  }

  // 高阶函数
  function wrap(WrappedComponent){

    HOCComponent.WrappedComponent = WrappedComponent;

    // 返回创建好的高阶函数
    return HOCComponent;

  }

  // 返回需要封装的高阶函数
  return wrap;

}


function combineReducers(reducers){

  // 获取所有函数键
  const reducerKeys = Object.keys(reducers);

  // 返回封装之后的函数
  return function finalReducer(state = {}, action){

    // 最终的状态
    const nextState = {};

    // 依次对于 Reducer 进行处理
    for(const key of reducerKeys){
      // 获取 reducer
      const reducer = reducers[key];

      // 获取状态树中的子对象
      const stateByKey = state[key];

      // 执行 reduce 转换操作
      const nextStateByKey = reducer(stateByKey, action);

      // Redux 需要避免状态空，进行异常检测
      if(typeof nextStateByKey === "undefined"){
        throw new Error("Invalid Reducer");
      }

      // 将新的状态对象挂载
      nextState[key] = nextStateByKey;

    }

    return nextState;

  }

}



function clientCacheMiddleware({ maxAge=3600 * 24 * 365 }){
  return function(req, res, next){
    res.setHeader("Cache-Control",`max-age=${maxAge}`);

    next();
  };
}
```
