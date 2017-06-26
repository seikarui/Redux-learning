
###What is redux-saga? What is used for?
	
	When developing front-ends, handling asynchronous behavior is always bit of a challenge. Redux-saga is one solution for these problems.It is a Redux middleware for handling side effects. In Function Programming, the things like Asynchronous and impure are called `side effect`.


The author of redux-saga said that:

	I want to emphasize that you don't actually have to go through academic papers and backend concepts in order to use redux-saga. It's sufficient to know that a saga is a piece of code which runs in the background, watch for dispatched actions, may perform some async calls (or synchronous impure calls like browser storage) and can dispatch other actions to the store.
	
	The redux-saga is implemented using the generator functions.(Which shared by dongying last test). Unlike other functions which run completion and return a value. The generator can be easily paused and resumed on demand and can return multi values.
	
	
###The basic usage.

####Send Async request using saga:

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

#### Using redux-thunk

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


####Coordinator concurrent tasks.

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



####Easy to test




All operations inside sagas are yielded as plain JavaScript objects, which then get executed by the middleware. This makes it very easy to test the business logic inside the saga. You simply iterate over the generator and test the yielded sequence of objects by a simple deepEqual.

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












How to implement in our project?