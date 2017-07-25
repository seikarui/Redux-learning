
## What is redux-saga? What is used for?

>When developing front-ends, handling asynchronous behavior is always bit of a challenge. Redux-saga is one solution for these problems. It is a Redux middleware for handling side effects. In Function Programming, the things like Asynchronous and impure are called `side effect`.


The author of redux-saga said that:

>I want to emphasize that you don't actually have to go through academic papers and backend concepts in order to use redux-saga. It's sufficient to know that a saga is a piece of code which runs in the background, watch for dispatched actions, may perform some async calls (or synchronous impure calls like browser storage) and can dispatch other actions to the store.


>The redux-saga is implemented using the generator functions(Which shared by dongying last test). Unlike other functions which run completion and return a value. The generator can be easily paused and resumed on demand and can return multi values.

## Generator

For common functions:
> once the function starts running, it will always run to completion before any other JS code can run.

For Generator:
> It may be paused once or may times, and allow other code to run during these paused periods. it pauses itself when it comes across a `yield` commands, and return the value of the yield expression. Although it stopped by itself, but it cannot resume on its own. It must be restarted by an external control.

[example code](https://babeljs.io/repl/#?babili=false&evaluate=true&lineWrap=false&presets=es2015%2Creact%2Cstage-1%2Cstage-2&targets=&browsers=&builtIns=false&debug=false&code_lz=GYVwdgxgLglg9mABAKkRAhgG0wCgJQDeAUIogJ4wCmmAJogIwDcJ5VtiATM6RdXQMzMAvkSIQEAZyiJ0iALxosuPM3FgJcTJQB0mOAHMc6bWEoAPKPhVjJmnXsPHTFq6ttbdBoyfOW81tQ0PB29nP2sgA)

```
function * call(){
  yield 1;
  yield 2;
  yield 3;
}

const a = call();
console.log(a.next());
console.log(a.next());
console.log(a.next());
console.log(a.next());
```

## The basic glossary.

1.Effect:

> An effect is an plain Javascript Object containing some instructions, which to be excute by the saga.

There are lots of Effect functions in saga:

take

> Creates an Effect description that instructs the middleware to wait for a specified action on the Store. The Generator is suspended until an action that matches pattern is dispatched.

```
yield take('FETCH_REQUEST')
```

put

> Creates an Effect description that instructs the middleware to dispatch an action to the Store. This effect is non-blocking and any errors that are thrown downstream (e.g. in a reducer) will not bubble back into the saga.

```
yield put({
    type: 'FETCH_SUCCESS',
    data
  })
```
call

> Creates an Effect description that instructs the middleware to call the function fn with args as arguments.
call(func, ...args)

fork

> Creates an Effect description that instructs the middleware to perform a non-blocking call on fn.
eg:
```
fork(func, param)
```

cancel

> Creates an Effect description that instructs the middleware to cancel a previously forked task.

race

> Creates an Effect description that instructs the middleware to run a Race between multiple Effects (this is similar to how Promise.race([...]) behaves).
```
 yield race({
    posts: call(fetchApi, '/posts'),
    timeout: call(delay, 1000)
  })
```


2.Watcher/Worker pattern.

>The watcher: will watch for dispatched actions and fork a worker on every action

>The worker: will handle the action and terminate

```
function* watcher() {
  while (true) {
    const action = yield take(ACTION)
    yield fork(worker, action.payload)
  }
}

function* worker(payload) {
  // ... do some stuff
}
```


###### Send Async request using saga:

```
import { take, fork, call, put } from 'redux-saga/effects';

// The watcher: watch actions and coordinate worker tasks
function* watchFetchRequests() {
  while(true) {
    const action = yield take('FETCH_REQUEST');

    yield fork(fetchUrl, action);
  }
}

// The worker: perform the requested task
function* fetchUrl(action) {
  const data = yield call(fetch, action.url);

  yield put({
    type: 'FETCH_SUCCESS',
    data
  });
}
```

UI Components never invoke the tasks themselves, instead they always dispatch plain object actions to notify that something happened in the UI.


```
dispacth({
  type: 'FETCH_REQUEST',
  url: /* ... */}
);

```

###### Using redux-thunk

```
function fetchUrl(url) {
  return (dispatch) => {
    dispatch({
      type: 'FETCH_REQUEST'
    });

    fetch(url).then(data => dispatch({
      type: 'FETCH_SUCCESS',
      data
    }));
  }
}

//Call the func in the UI, and then dispatch.
dispatch(
  fetchUrl(url)
)
```

###### Why we select saga?

Comparation between redux-thunk and redux-saga.

1) Handle the concurrent work flow.

When the User click the button multiply times. In thunk, the multi request will be sent out, we cannot control the flow. But we can control the work flow in redux-saga.

For example, if we want to cancel the request when new request coming. We can try this:

```
import { take, fork, cancel, call, put } from 'redux-saga/effects';

function* watchFetchRequests() {
  let currentTask;

  while(true) {
    const action = yield take('FETCH_REQUEST');

    if(currentTask) {
      yield cancel(currentTask);
    }

    currentTask = yield fork(fetchUrl, action);
  }
}

```


The above function is implement by the saga itself, we can use `takeLatest` to replace.

```
function* watchFetchRequests() {
  yield takeLatest("FETCH_REQUEST", fetchUrl);
}
```

And also, we have the `takeEvery` Effect, to receive and send every request.



###### Easy to test


All operations inside sagas are yielded as plain JavaScript objects, which then get executed by the middleware. This makes it very easy to test the business logic inside the saga. You simply iterate over the generator and test the yielded sequence of objects by a simple deepEqual.

For example:

```
import { put, take} from 'redux-saga/effects';

function * fetchData(){
  const action = yield take('FETCH_DATA');
  yield put('FETCH_SUCCESS');
}

```

the result of yield take:

{ '@@redux-saga/IO': true, TAKE: { pattern: 'FETCH_DATA' } }

the result of yield put:

{ '@@redux-saga/IO': true,
     PUT: { channel: null, action: 'FETCH_SUCCESS' } }



```
function* fetchUrl(action) {
  const data = yield call(fetch, action.url);

  yield put({
    type: 'FETCH_SUCCESS',
    data
  });
}

//////test

const generator = fetchUrl();

assert.deepEqual(
  generator.next().value,
  take('FETCH_RESULT')
);

// we can easily mock the result of the `take('FETCH_RESULT')` call
const mockAction = {
  url: 'some url'
};

// and inject the result back into the Generator
assert.deepEqual(
  generator.next(mockAction).value,
  take('FETCH_SUCCESS')
);
```
###### Error handling

We can catch errors inside the Saga using the familiar try/catch syntax.

```
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

// ...

function* fetchProducts() {
  try {
    const products = yield call(Api.fetch, '/products')
    yield put({ type: 'PRODUCTS_RECEIVED', products })
  }
  catch(error) {
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
  }
}
```

test:

```
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)

// create a fake error
const error = {}

// expects a dispatch instruction
assert.deepEqual(
  iterator.throw(error).value,
  put({ type: 'PRODUCTS_REQUEST_FAILED', error }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_REQUEST_FAILED', error })"
)

```

###### How to use saga in redux?

In order to run our Saga, we need to:

1. create a Saga middleware with a list of Sagas to run (so far we have only one helloSaga)
2. connect the Saga middleware to the Redux store


```
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

// ...
import { rootSaga } from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
sagaMiddleware.run(rootSaga)

const action = type => store.dispatch({type})

```


## Some use case

1. When you want to run some logic after all kinds of actions been dispatched.

```
import { select, takeEvery } from 'redux-saga/effects'

function* watchAndLog() {
  yield takeEvery('*', function* logger(action) {
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  })
}

```
2. Condition run the some logic.

```
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for (let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}

```

3. Convey the expected action sequence.

```
function* loginFlow() {
  while (true) {
    yield take('LOGIN')
    // ... perform the login logic
    yield take('LOGOUT')
    // ... perform the logout logic
  }
}

```


4. Run tasks in parallel

```
import { all, call } from 'redux-saga/effects'

// correct, effects will get executed in parallel
const [users, repos]  = yield all([
  call(fetch, '/users'),
  call(fetch, '/repos')
])
```


5. Start a race on multi tasks.

```
import { race, take, put } from 'redux-saga/effects'

function* backgroundTask() {
  while (true) { ... }
}

function* watchStartBackgroundTask() {
  while (true) {
    yield take('START_BACKGROUND_TASK')
    yield race({
      task: call(backgroundTask),
      cancel: take('CANCEL_TASK')
    })
  }
}
```
6. Cancel

```
import { take, put, call, fork, cancel, cancelled } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { someApi, actions } from 'somewhere'

function* bgSync() {
  try {
    while (true) {
      yield put(actions.requestStart())
      const result = yield call(someApi)
      yield put(actions.requestSuccess(result))
      yield call(delay, 5000)
    }
  } finally {
    if (yield cancelled())
      yield put(actions.requestFailure('Sync cancelled!'))
  }
}

function* main() {
  while ( yield take(START_BACKGROUND_SYNC) ) {
    // starts the task in the background
    const bgSyncTask = yield fork(bgSync)

    // wait for the user stop action
    yield take(STOP_BACKGROUND_SYNC)
    // user clicked stop. cancel the background task
    // this will cause the forked bgSync task to jump into its finally block
    yield cancel(bgSyncTask)
  }
}
```
