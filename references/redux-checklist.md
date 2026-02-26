# Redux Checklist

## State Shape & Design

### State Normalization

- **Denormalized nested state**: Deep object hierarchies that are hard to update
  ```javascript
  // Bad: Deeply nested, hard to update user.orders[0].items[0].price
  {
    users: {
      byId: {
        1: {
          name: 'John',
          orders: [
            {
              id: 'o1',
              items: [{ id: 'i1', price: 100 }]
            }
          ]
        }
      }
    }
  }
  
  // Good: Normalized with references
  {
    users: { byId: { 1: { name: 'John' } } },
    orders: { byId: { 'o1': { userId: 1 } } },
    items: { byId: { 'i1': { price: 100, orderId: 'o1' } } }
  }
  ```
- **Storing arrays when byId object would be better**: Linear search O(n) instead of O(1) lookup
- **Mixed normalized and denormalized data**: Inconsistency and update bugs
- **Storing derived data in state**: Calculated values that should be in selectors instead
  ```javascript
  // Bad: Derived data in state
  { items: [...], total: 500, itemCount: 3 }
  
  // Good: Computed in selector
  export const selectTotal = state =>
    state.items.reduce((sum, item) => sum + item.price, 0)
  ```

### State Ownership

- **Multiple sources of truth**: Same data stored in multiple places
- **Unclear state ownership**: No clear which reducer owns what
- **Mixing domain and UI state**: Business logic coupled to view state
  ```javascript
  // Better: Separate concerns
  {
    entities: { users, posts },      // Domain state
    ui: { selectedPostId, isModal }, // UI state
    async: { fetchStatus, error }    // Async state
  }
  ```
- **Storing entities not owned by this reducer**: Cross-cutting concerns

### Questions to Ask
- "Can I look up any entity by ID in O(1)?"
- "Where is the source of truth for this data?"
- "Is this data derived or fundamental?"

---

## Reducers & Purity

### Reducer Purity

- **Mutating state directly**: Using `.push()`, assignment, or spread on same object
  ```javascript
  // Bad: Direct mutation
  state.items.push(action.payload)
  
  // Good: Return new state
  return {
    ...state,
    items: [...state.items, action.payload]
  }
  ```
- **Mutating action payload**: Modifying `action.payload` instead of creating new objects
- **Non-deterministic reducers**: Using `Date.now()`, `Math.random()` in reducer
- **Side effects in reducers**: API calls, imports, external I/O
- **Async operations in reducer**: Using `.then()` or `async/await`
  ```javascript
  // Bad: Async in reducer
  const reducer = (state, action) => {
    if (action.type === 'FETCH') {
      fetch(url).then(data => state.result = data) // WRONG!
    }
  }
  
  // Good: Middleware handles async
  ```

### Default/Initial State

- **Missing default case in reducer**: Returns `undefined` instead of state
  ```javascript
  // Good: Always return something
  export default function itemsReducer(state = [], action) {
    switch(action.type) {
      case 'ADD': return [...state, action.payload]
      default: return state
    }
  }
  ```
- **Initial state as null**: Null checks required everywhere
- **Not initializing user state on login**: Stale data from previous user

### Update Patterns

- **Not using Immer or helper for deep updates**: Complex immutable logic
  ```javascript
  // Better: Use Immer (Redux Toolkit)
  const itemsSlice = createSlice({
    name: 'items',
    initialState: [],
    reducers: {
      updateItem: (state, action) => {
        const item = state.find(i => i.id === action.payload.id)
        if (item) item.price = action.payload.price // Immer handles immutability
      }
    }
  })
  ```
- **Nested spreads for deep updates**: Boilerplate and error-prone
- **Not validating action payload types**: Type errors at runtime

### Questions to Ask
- "Could this reducer cause mutation bugs?"
- "Does this return a new state or the same reference?"
- "What happens on first render when state is undefined?"

---

## Actions & Action Creators

### Action Design

- **Non-descriptive action types**: Using string that could collide
  ```javascript
  // Better: Namespace action types
  const FETCH_USERS_REQUEST = 'users/FETCH_REQUEST'
  const FETCH_USERS_SUCCESS = 'users/FETCH_SUCCESS'
  const FETCH_USERS_ERROR = 'users/FETCH_ERROR'
  ```
- **Over-dispatching actions**: Too many actions for simple change
- **Actions with complex payloads**: Serialization issues, hard to debug
- **Not normalizing action payloads**: Nested objects passed as-is

### Async Actions

- **Not handling loading/error states**: No indication of async status
  ```javascript
  // Good: Track async state
  {
    data: null,
    loading: false,
    error: null
  }
  ```
- **Race conditions with async**: Older request overwrites newer one
  ```javascript
  // Bad: Can race
  fetchUsers().then(data => dispatch(setUsers(data)))
  
  // Good: Cancel previous or use flag
  useEffect(() => {
    let cancelled = false
    fetchUsers().then(data => !cancelled && dispatch(setUsers(data)))
    return () => { cancelled = true }
  }, [])
  ```
- **Not wrapping thunks properly**: Middleware not handling errors
- **Thunks returning promises but not awaited**: Unreliable async flow

### Questions to Ask
- "Is this action descriptive enough for time-travel debugging?"
- "Could two instances of this action cause inconsistency?"
- "What if this async action is dispatched twice in quick succession?"

---

## Selectors & Memoization

### Selector Performance

- **Creating selectors inline**: New function every render, no memoization
  ```javascript
  // Bad: Selector created every time
  const users = state.users.filter(u => u.active)
  
  // Good: Memoized selector
  export const selectActiveUsers = state =>
    state.users.filter(u => u.active)
  ```
- **Not using `reselect`**: Selectors re-compute on every state change
  ```javascript
  // Bad: Recomputes even if users unchanged
  export const selectUserCount = state => state.users.length
  
  // Good: Memoized (with reselect)
  import { createSelector } from 'reselect'
  export const selectUserCount = createSelector(
    state => state.users,
    users => users.length
  )
  ```
- **Selectors with uncached derived data**: Complex calculations not memoized
- **Component subscribing to large state slice**: Re-renders on unrelated updates

### Selector Composability

- **Not composing selectors**: Duplicated logic in multiple selectors
  ```javascript
  // Good: Selectors build on each other
  const selectUsersState = state => state.users
  const selectAllUsers = createSelector(
    selectUsersState,
    users => users.byId ? Object.values(users.byId) : []
  )
  const selectActiveUsers = createSelector(
    selectAllUsers,
    users => users.filter(u => u.active)
  )
  ```
- **Selectors not parameterized**: Hard to reuse for different inputs
  ```javascript
  // Good: Factory function for parameterized selectors
  export const makeSelectUserById = userId =>
    createSelector(
      state => state.users.byId?.[userId],
      user => user
    )
  ```

### Questions to Ask
- "Will this selector recompute unnecessarily?"
- "Can this selector be created once and reused?"
- "Are selectors properly memoized with reselect?"

---

## Middleware & Effects

### Side Effects Management

- **Side effects in action creators**: Network calls, storage writes not in middleware
  ```javascript
  // Bad: Side effect in action creator
  export const fetchUser = (id) => {
    fetch(`/users/${id}`).then(...)
  }
  
  // Good: Thunk middleware handles async
  export const fetchUser = (id) => async (dispatch) => {
    dispatch(fetchUserRequest())
    try {
      const data = await fetch(`/users/${id}`).json()
      dispatch(fetchUserSuccess(data))
    } catch (e) {
      dispatch(fetchUserError(e.message))
    }
  }
  ```
- **Thunk vs Saga vs Observable**: Choosing wrong pattern for use case
- **Middleware not cancelling previous effects**: Resource leaks, stale data
- **Not handling errors in middleware**: Silent failures
- **Race conditions between effects**: Multiple inflight mutations to same data

### Middleware Ordering

- **Incorrect middleware order**: Logger runs before thunk processor
  ```javascript
  // Good order: Thunk first, then logger, then router
  const store = createStore(
    rootReducer,
    applyMiddleware(thunk, logger, routerMiddleware)
  )
  ```
- **Custom middleware not compatible with expected types**: Type errors

### Questions to Ask
- "Where should this side effect live (component, middleware, thunk)?"
- "What happens if this effect is triggered twice?"
- "Is the middleware order correct for all dependencies?"

---

## Redux DevTools & Debugging

### Development Tools Usage

- **Not using Redux DevTools**: Missing time-travel debugging capability
- **Not loading initial state from persisted store**: Fresh hydration every reload
- **Not enabling DevTools in production build**: Performance hit and bundle size
  ```javascript
  // Good: Conditional DevTools
  const composeEnhancers =
    typeof window === 'object' && window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__
      ? window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__
      : compose
  ```
- **Skipping actions from DevTools history**: Missing context for debugging

### Logging & Monitoring

- **Not logging important state changes**: Difficult to debug issues in production
- **Logging sensitive data to Redux state**: PII/secrets visible in DevTools
- **Not tracking action timing**: Performance issues hard to diagnose
- **Missing error boundaries around reducer logic**: Crashes not caught

### Questions to Ask
- "Can I time-travel to this action and see the result?"
- "Are sensitive fields excluded from logs?"
- "Is this action clearly visible in DevTools history?"

---

## Store Configuration & Enhancement

### Store Creation

- **Missing Redux Toolkit**: Using vanilla Redux instead of modern best practices
  ```javascript
  // Modern: Redux Toolkit (Immer + improved config)
  import { configureStore } from '@reduxjs/toolkit'
  const store = configureStore({
    reducer: {
      items: itemsSlice.reducer,
      users: usersSlice.reducer
    }
  })
  ```
- **Not handling store hydration**: Async data not loaded on startup
- **Missing timestamp or versioning**: State migrations impossible
- **Store persisted but not hydrated**: Initial state mismatch

### Enhancing Store

- **Custom middleware poorly integrated**: Hard to debug, not composable
- **Too much middleware**: Performance hit and complex logic flow
- **Not handling unhandled errors in middleware**: Crashes or silent failures

### Questions to Ask
- "Is this store configuration compatible with React 18 Concurrent Rendering?"
- "How is persisted state hydrated on app startup?"
- "Are there any circular dependencies in middleware?"

---

## Common Patterns & Anti-patterns

### Containers vs Presentational

- **All components connected to Redux**: No separation of concerns
- **Mutating props from Redux**: Props should be immutable
- **Not memoizing connected components**: Re-renders on unrelated state changes
  ```javascript
  // Good: Memoize after connect
  export default connect(mapStateToProps)(React.memo(MyComponent))
  ```

### Redux Subscribers & Unsubscription

- **Not unsubscribing from store**: Memory leaks in effects
  ```javascript
  // Good: Cleanup subscription
  useEffect(() => {
    const unsubscribe = store.subscribe(handleStoreChange)
    return unsubscribe
  }, [])
  ```
- **Multiple subscriptions to same state slice**: Redundant renders

### Combining Multiple Stores

- **Creating multiple Redux stores**: Defeats purpose of single source of truth
- **Mixing Redux with useContext**: Confusing data flow
- **Not using combineReducers**: Monolithic reducer function hard to maintain

### Questions to Ask
- "Is this component doing too much or too little with Redux?"
- "Are subscriptions properly cleaned up?"
- "Could this data be in React state instead?"

---

## Redux with TypeScript

### Type Safety

- **Not typing action payload**: Runtime errors from shape mismatches
  ```javascript
  // Good: Typed actions
  interface SetUserAction {
    type: 'SET_USER'
    payload: {
      id: string
      name: string
    }
  }
  type UserAction = SetUserAction | AnotherAction
  ```
- **Not typing reducer**: State type not enforced
- **Action types as magic strings**: Typos cause silent failures
- **Not using Redux Toolkit types**: Manual setup error-prone
  ```javascript
  // Good: Redux Toolkit with TypeScript
  const itemsSlice = createSlice({
    name: 'items',
    initialState: [] as Item[],
    reducers: {
      addItem: (state, action: PayloadAction<Item>) => {
        state.push(action.payload)
      }
    }
  })
  ```

### Selector Typing

- **Selectors without return type**: Hard to use in typed code
- **Parameterized selectors not properly typed**: Runtime type errors possible

### Questions to Ask
- "Are all action payloads typed?"
- "Is the reducer state type enforced?"
- "Are selectors properly typed for consumers?"
