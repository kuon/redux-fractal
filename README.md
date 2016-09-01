# redux-fractal
Local component state &amp; actions in Redux.

Provides the means to hold up local component state in redux state,  to dispatch locally scoped actions and to react to global ones.

What Redux fractal offers is a Redux private store for each component with the notable difference that the component state is actually help up in your
app's state atom, so all global and components ui state live together.

The unique and powerful approach consists in the fact that it allows you to
use the built-in Redux createStore, combineReducers and others for defining the shape and managing the UI ,state with great benefits:
- All state ( either application state or UI state) is help up in your app single state atom
- Component state updates are managed via reducers just like your global app state which means predictability and easy testability
- Reducers used for local state aren't aware that they are used for managing  state for a component instance. As such they can be easily re-used across components or even used for managing global state.
- You can have per component middleware. This opens up interesting possibilities like locally scoped sagas(eg redux-saga), intercepting and handling of all actions generated by a component before they reach the component's UI store.
- By default a component intercepts in reducer only actions generated by itself but it;s easy to enable intercepting global actions or actions generated by other components.

It's easy to get started using redux-fractal
## Installation

```console
$ npm install --save redux-fractal
```
## Usage
### Importing the local reducer into global store
Add the local reducer to the redux store under the
'local' reducer key;
```js
    import { localReducer } from 'redux-fractal';
    const store = createStore(combineReducers({
        local: localReducer,
        myotherReducer: myotherReducer
    }))
```
### Adding the `local` HOC to components maintaining UI state
Decorate the components that hold ui state( transient state, scoped to that very specific component ) with the 'local' higher order component and provide a mandatory, globally unique key for your component and a createStore method.

The key can be generated based on props or a static string but it must be unique and be stable between re-renders. Basically it should follow exactly
the same rules as the React component 'key' but be globally unique.
```js
    import { local } from 'redux-fractal';
    import { createStore } from 'redux';

    const CompToRender = local({
        key: 'myDumbComp',
        createStore: (props) => {
            return createStore(rootReducer, { filter: true, sort: props.sortOrder })
        }
    })(Table);
```
### Defining root reducer for the private store
Define a root reducer that will intercept own dispatched functions( by default only actions dispatched from the wrapped component, but the 'local' higher order component can be easily configured to  intercept actions from global Redux instance or generated by other components):
```js
const rootReducer = (state = { filter: null, sort: null, trigger: '', current: '' }, action) => {
     switch(action.type) {
         case 'SET_FILTER':
            return Object.assign({}, state, { filter: action.payload });
         case 'SET_SORT':
            return Object.assign({}, state,
                { sort: action.payload });
         case 'GLOBAL_ACTION':
            return Object.assign({}, state, { filter: 'globalFilter' });
        case 'RESET_DEFAULT':
           return Object.assign({}, state, { sort: state.sort+'_globalSort' });
         default:
            return state;
     }
};
```
Note that the reducer is like any other ordinary reducer used in a Redux app. The difference is  that it manages and controls the state transitions for a certain component state.

In fact, you can use whatever method of combining reducers you use for your app with no exceptions, also for the individual components:
```js
import { combineReducers, createStore } from 'redux';
const rootReducer = combineReducers({
    filter: filterReducer,
    sort: sortReducer
});
local({
    key: (props) => props.tableID,
    createStore: (props) => {
        return createStore(rootReducer, componentInitialState);
    }
})
```
## Accessing local state and dispatching local actions
The well know `mapStateToProps` and `mapDispatchToProps` familiar from react-redux 'connect' are available having the very same signatures.
In fact , internally, redux-fractal uses the connect function from 'react-redux' to connect the component to it's private store.

The difference is, that you get only the component's state in `mapStateToProps` as opposed to the entire app state and the `dispatch`
function in `mapDispatchToProps` dispatches an action tagged with the component key as specified in the HOC config.
Both `mapStateToProps` and `mapDispatchToProps` are completely optional, define them if you need them.
### Mapping component state to props
Beware that the components wrapped in 'local' HOC do not update when global state changes but only when their own state changes. These components are effectively connected to their own private store.
Of course, you can also connect them to the global store using standard 'connect' function.
```js
    local({
        key: 'mycomp',
        createStore: (props) => createStore(rootReducer, initialState),
        mapStateToProps: (componentState, ownProps) => {
            // Get component state from it's private store and
            // component own props.
            // You must return an object containing the keys that will become props to the component just like in react redux 'connect'
            return {
                filter: getFilter(componentState)
            }
        },

    })
```
By default, if no `mapStateToProps` is defined, then all keys from the component's state become
individual props in the wrapped component:
```js
    local({
        key: 'mycomp',
        createStore: (props) => createStore(rootReducer, { filter: 'all', sort: 'asc' }),
        mapStateToProps: (componentState, ownProps) => {
            // Get component state from it's private store and
            // component own props.
            // You must return an object containing the keys that will become props to the component just like in react redux 'connect'
            return {
                filter: getFilter(componentState)
            }
        },

    })(Table)
```
In the Table you now have access to the 'filter' state via props.filter.
### Dispatching local actions
Local actions can be dispatched by defining a 'mapDispatchToProps' function.
Note that local dispatches can be caught by the component's own reducer AND by global, application wide reducers.
```js
    import { updateSearchTerm } from ''
    local({
        key: 'mycomp',
        createStore: (props) => createStore(rootReducer, initialState),
        mapDispatchToProps: (dispatch) => {
            // ALL actions dispatched via 'dispatch' function above have the component key tagged to the action.
            // You can see that by inspecting action.meta.triggerComponentKey in redux dev tools.
            // All local actions are dispatched also on the global store
            // You must return an object containing the keys that will become props to the component just like in react redux 'connect'
            return {
                onFilter: (term) => dispatch(updateSearchTerm(term))
            }
        },

    })
```

These actions can be caught in any global reducer and by default only in the originating component reducer.
One can inspect the originating component by looking at `action.meta.triggerComponentKey` to get the component's key that dispatched the action.

```js
import { combineReducers, createStore } from 'redux';
const rootReducer = combineReducers({
    filter: filterReducer,
    sort: sortReducer
});
local({
    id: "mygreattable",
    createStore: (props) => {
        return createStore(rootReducer, componentInitialState);
    },
    mapStateToProps: (componentState, ownProps) => ({
        filter:  getTableFilter(componentState)
    }),
    mapDispatchToProps: (localDispatch) =>({
        onFilter: (term) => localDispatch(updateSearchTerm(term))
    })
})
```
### Reacting to globally dispatched actions or actions dispatched from other components
By default your component will not react to when other components dispatch
local actions or when something is being dispatched in the global store.
You can change that using `filterGlobalActions` which must return either true if the action
can be forwarded to the component's store or false otherwise.
Locally dispatched actions are ALWAYS forwarded to the corresponding component reducer.
You need to define only if you care in updating the component state based on actions happening globally or in other
components.

```js
local({
    id:  (props) => props.itemID,
    creat
    filterGlobalActions: (action) => {
        // Any logic to determine if the actions should be forwarded
        // to the component's reducer. By default none is except those
        // originated by component itself
        const allowedActions = ['RESET_FILTERS', 'CLEAR_SORTING'];
        return allowedActions.indexOf(action.type) !== -1;
    }
})
```
Now any RESET_FILTERS or CLEAR_SORTING global actions or originated by other components will be allowed.
You have lots of flexibility with this method to react when a component updates it's UI state.
Crazy example: when the sorting from one component changes all dropdowns from another component should close:
 ```js
 local({
     id:  'dropdownsContainer',
     createStore: (props) => {
        return createStore((state = {isClosed: false}, action) => {
            switch(action.type) {
                case SET_SORT:
                    return Object.assign({}, state, { isClosed: true });
                break;
                default:
                    return state;
            }
        });
     },
     filterGlobalActions: (action) => {
         // This component is interested in updating it's state
         // when things happen in the 'mygreattable' component, in this
         // case when sorting changes
         const allowedActions = ['SET_SORT'];
         return allowedActions.indexOf(action.type) !== -1 && actions.meta.triggerComponentKey === 'mygreattable';
     }
 })
 ```
## Local middleware
Since Redux-fractal relies on redux for manging the component state and offers a private store for the component
you can define middleware for the private component store using the exact same approach as you do when setting up
the application's store.
This opens up some interesting possibilities:
- Use locally scoped sagas that get destroyed on component unmount(eg using [Redux saga](https://github.com/yelouafi/redux-saga) ). These sagas will have access only to the component's private store( so all yield select() would actually return the component's state) and also be able to react to all local actions and actions allowed by `filterGlobalActions`.
- Apply per component throttling/debouncing of actions using middleware's such as redux-batch-subscribers
- Transform the actions originating in components of a certain type
- Track at all times the component that initiated for eg a server request, validations etc by using locally
scoped redux-thunk middleware and dispatching thunks( in which case the 'dispatch' and `getState` functions will be the ones from the component local store)
- Re-use middleware that you use on the global store on the component's private store

### Local Redux saga example
First install redux-saga as describe here [Redux saga](https://github.com/yelouafi/redux-saga).
Define a saga somewhere( eg in sagas.js):
```js
function* fetchUser(action) {
   const compState = yield select();
   try {
      const user = yield call(Api.fetchUser, action.payload.userId);
      yield put({type: "USER_FETCH_SUCCEEDED", user: user});
   } catch (e) {
      yield put({type: "USER_FETCH_FAILED", message: e.message});
   }
}
export default function* mySaga() {
  yield* takeLatest("USER_FETCH_REQUESTED", fetchUser);
}
```
Then wrap a component in the 'local' HOC:
```js
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

import reducer from './reducers'
import mySaga from './sagas'
    const componentRootReducer = (state = { user: {} }, action) => {
         switch(action.type) {
             case USER_FETCH_SUCCEEDED:
                return Object.assign({}, state, { user: action.payload });
             default:
                return state;
         }
    };
    local({
        key: 'formContainer',
        createStore: (props) => {
            // create the saga middleware
            const sagaMiddleware = createSagaMiddleware();
            const store = createStore(
                componentRootReducer,
                applyMiddleware(sagaMiddleware)
            );
            sagaMiddleware.run(mySaga)
            return { store: store, cleanup: () => sagaMiddleware.cancel() };
        },
        mapDispatchToProps: (dispatch) => {
            onFetchUser: (userId) => dispatch({type: "USER_FETCH_REQUESTED", payload: userId })
        }
    })
```
All of the `put()` effects dispatch a local action.
All of the `select()` effects return parts of the component's state.
All of the `take()` effects react to the very same action's that local reducers are allowed to react:
- Locally dispatched actions
- Global or other component actions allowed by `filterGlobalActions` function.

It's important to note that by default component stores do not contain middleware, just as
the global Redux store doesn't contain it by default and middleware needs to be added to it.
This means for example that implictly you can only dispatch plain objects. To dispatch functions, promises etc
configure the private component state with the needed middleware( eg redux-thunk etc )
## Recipes
### I need access to global application state in mapStateToProps of the `local()` component

Access to only the component's internal state in `mapStateToProps` is a conscious design decision.
If you need data from the global state and need to update the component when that data changes you can
wrap the component returned by `local` in `connect()`.
That way you will be able to pass data from the global store into the component returned by `local` via props.
```js
    import { connect } from 'react-redux';
    import { local } from 'redux-fractal';
    import { createStore, compose } from 'redux';
    const wrapper = compose(
        connect((state) => ({
            userSettings: state.UserSettingsReducer.settings
        })),
        local({
            key: 'comp',
            createStore: (props) => {
                const filterVal = props.userSettings.defaultFilter;
                return createStore(componentRootReducer, { filter: filterVal });
            }
        });
    export default wrapper(MyComponent);
```
### I need to keep the local state even if the component gets unmounted and restore it when it gets mounted again
In that case you can use the `persist` flag on the `local` HOC. When the flag is set to `true`
you will get the existing component state, when the component re-mounts, as the second parameter
to the `createStore` function.
```js
import { local } from 'redux-fractal';
import { createStore } from 'redux';
const wrapper = local({
        key: 'comp',
        createStore: (props, existingState) => {
            const filterVal = 'search';
            return createStore(componentRootReducer, existingState || { filter: filterVal });
        },
        persist: true
    });
export default wrapper(MyComponent);
```
If there are multiple instance of MyComponent, using `persist` as shown above will persist the state of ALL instances.
You can have fine grained control over which component instances get persisted and which do not
by defining `persist` as a function receiving the component props.
```js
const HOC = local({
    key: (props) => props.itemId,
    createStore: (props, existingState) => {
        return createStore(
            rootReducer,
            existingState || { filter: true }
        );
    },
    // Any logic depending on props here to decide if the component state should be persisted
    persist: (props) => props.keepState
});
const ConnectedItem = HOC(Item);

const App = (props) => {
    return (
            <div>
               <ConnectedItem itemId={'a'} keepState={true} />
               <ConnectedItem itemId={'b'} keepState={false} />
            </div>
    );
}
```
In the above example only the state of component with itemId 'a' will be kept in the global
state after the component unmounts.

### I need to read a component's state somewhere else: eg in thunk action creators, sagas, in `mapStateToProps`
All the local components state is available at `state.local[key]` where `key` is the key for the component
as return by the `key` property of the `local` HOC.

## TODO (Help wanted)
 - Write additional tests
 - Verify server side rendering
 - Verify behaviour with custom middleware
