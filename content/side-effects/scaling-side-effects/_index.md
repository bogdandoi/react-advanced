+++
title = "Scaling Side Effects"
date = 2020-05-04T16:06:56+01:00
weight = 2
draft = false
+++

One of the most complex aspects you encouter in side effect management is dealing with authentication and authentication
storage on the client side. In order to make sure you are following best security guidelines the token needs to be
stored as `http-only`. That means that you will not be able to access it from the client side code.

There are a few pieces of the data available that can be resolved by authenticating with your api. Because of the
`http-only` approach with our JWT token the way to extract it is by making calls to a `check-token` endpoint which is
part of our auth flow.

In order to avoid code duplication we will use a render prop pattern to create a container for our components that is
rendered before its children and populates the store with the slice of user information retrieved from the auth checker
endpoint.

The container should look something like this `packages/goodreads/src/containers/auth-checker/index.js`
```javascript
import { Component } from 'react'
import { checkAuth } from './actions'
import { connect } from 'react-redux'

class AuthCheck extends Component {
  componentDidMount() {
    const { dispatch } = this.props
    return dispatch(checkAuth())
  }

  render() {
    const { children } = this.props
    return children
  }
}

const mapStateToProps = ({ username }) => {
  return { username }
}

export default connect(mapStateToProps)(AuthCheck)
```

#### Action flow
{{<mermaid>}}
stateDiagram
    [*] --> Component
    Component --> Action
    Action --> SagaMiddleware
    SagaMiddleware --> Redux
    Redux --> Component : rerender

    state SagaMiddleware {
        state watcherSaga {
            [*] --> api: request data
            api --> [*]: response data
        }
    }
    state Redux {
        [*] --> reducer
        reducer --> [*]
    }
{{</mermaid>}}

If you are familiar with redux and the way the functional cycle from the UI to redux and back works then this should be
fairly straightforward. The novelty is where the sagas fit into this flow and that is in the middleware section. Some
projects will use `thunks` to resolve data however when using thunks exclusively the actions will become quite unwieldy
do to callback hell.

Every request/response cycle has 3 actions associated (request start, request success, request failed). Sagas allow us
to have a very standard development pattern for dealing with an http request.

We already saw what a barebone component looks like. Actions are standard as well.

#### SagaMiddleware `packages/goodreads/src/store/index.js`

The sagaMiddleware needs to be created and also started for it to work
{{< highlight javascript "hl_lines=11 34" >}}
import { createStore, applyMiddleware } from 'redux'
import { composeWithDevTools } from 'redux-devtools-extension'
import createSagaMiddleware from 'redux-saga'
import rootSaga from './sagas'
import rootReducer from './rootReducer'

const composeEnhancers = composeWithDevTools({
  trace: true,
})

const sagaMiddleware = createSagaMiddleware()

const logger = ({ getState }) => (next) => (action) => {
  const console = window.console
  const prevState = getState()
  const returnValue = next(action)
  const nextState = getState()
  const actionType = String(action.type)
  const message = `action ${actionType}`
  console.log(`%c prev state`, `color: #9E9E9E`, prevState)
  console.log(`%c action`, `color: #03A9F4`, message)
  console.log(`%c next state`, `color: #4CAF50`, nextState)
  return returnValue
}

const middlewares = [sagaMiddleware, logger]

export default function configureStore(initialState = {}) {
  const store = createStore(
    rootReducer,
    initialState,
    composeEnhancers(applyMiddleware(...middlewares))
  )
  sagaMiddleware.run(rootSaga)
  return store
}
{{< /highlight >}}

The implementation of the api calls and the side effects is implemented closer to the store than if we were to use
thunks. This allows for a higher degree of decoupling.
```bash
   ▾ [ ]store/
     ▾ [ ]sagas/
         [ ]books.js
         [ ]index.js
         [ ]login.js
       [ ]api.js
       [ ]index.js
```

The rootSaga is a collation of all the sagas and is added to the middleware collection. Its task is to map specific
actions to side effect handlers that are called watchers.

{{< highlight javascript >}}
import { takeLatest } from 'redux-saga/effects'
import { CHECK_AUTH_STARTED } from '../../containers/auth-checker/actions'
...
import { watchAuthStatus } from './login'

export default function* rootSaga() {
  ...
  yield takeLatest(CHECK_AUTH_STARTED, watchAuthStatus)
}
{{< /highlight >}}

The watchers are the actuall functionality and this is where you will write the most of the functionality.

{{< highlight javascript >}}
import { call, put } from 'redux-saga/effects'
import * as api from '../api'
import {
  CHECK_AUTH_SUCCEEDED,
  CHECK_AUTH_FAILED,
} from '../../containers/auth-checker/actions'
...

export const watchAuthStatus = function* watchAuthCheck() {
  try {
    const { data } = yield call(api.checkToken)
    yield put({
      type: CHECK_AUTH_SUCCEEDED,
      payload: {
        username: data.username,
      },
    })
  } catch (e) {
    yield put({
      type: CHECK_AUTH_FAILED,
      payload: {
        error: 'Not authenticated',
      },
    })
  }
}
{{< /highlight >}}

The decoupling between the http requests and the actions allows you to combine the responses and avoid the callback hell
alltogether. IMO that is pretty cool especially when working with microservices or even services. If a component
requires data from multiple sources to be able to render itself then the ability to collate the data before sending it
on to the store is pretty much invaluable.

### We have reached a point where fetching data works

We have multiple API sources at this point:
- `/images` - fetch cover images for books
- `/meta` - fetch data for each book
- `/ratings` - fetch ratings for books
- `/auth/authenticate` - login
- `/auth/register` - creating new user account
- `/auth/check-token` - verify http token for validity whenever a component is mounted or receives new props

This is pretty great, however we will notice:
- our application is a really slow so performance needs some work still
- the requests are not synced correctly caused by the intermittent delays that are baked in intentionally into the API

## Case study (part I)

- Orchestrating the API requests so that the data is available more reliably for the `BookCard` component:
  we want to add functionality in the book saga to handle wating for data from `/images`, `/meta`, `/ratings` to be
  finished before attempting to render the book grid

## Bonus points
- Add actions, sagas and reducer cases for starting to read a book