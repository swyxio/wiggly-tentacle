---
id: hooks-reference
title: Hooks API Reference
permalink: docs/hooks-reference.html
prev: hooks-custom.html
next: hooks-faq.html
---

This page describes the APIs for the built-in Hooks in React. If you're new to Hooks, you might want to check out [the overview](/docs/hooks-overview.html) too.

- [Basic Hooks](#basic-hooks)
  - [`useState`](#usestate)
  - [`useEffect`](#useeffect)
  - [`useContext`](#usecontext)
- [Additional Hooks](#additional-hooks)
  - [`useReducer`](#usereducer)
  - [`useCallback`](#usecallback)
  - [`useMemo`](#usememo)
  - [`useRef`](#useref)
  - [`useAPI`](#useapi) (TODO)
  - [`useMutationEffect`](#usemutationeffect)
  - [`useLayoutEffect`](#uselayouteffect)

## Basic Hooks

### `useState`

```js
const [state, setState] = useState(initialState);
```

Returns a stateful value, and a function to update it.

During the initial render, the returned state (`state`) is the same as the value passed as the first argument (`initialState`).

The `setState` function is used to schedule an update. It accepts a new state value.

```js
setState(newState);
```

During a subsequent re-render of the parent component, the value returned by `useState` will be the most recent state after applying the updates.

#### Functional updates

If the new state is computed using the previous state, you can pass a function to `setState`. The function will receive the previous value, and return an updated value. Here's an example of a counter component that uses both forms of `setState`:

```js
function Counter({initialCount}) {
  const [count, setCount] = useState(initialCount);
  return (
    <>
      Count: {count}
      <button onClick={() => setCount(0)}>Reset</button>
      <button onClick={() => setCount(prevCount => prevCount + 1)}>+</button>
      <button onClick={() => setCount(prevCount => prevCount - 1)}>-</button>
    </>
  );
}
```

The "+" and "-" buttons use the functional form, because the updated value is based on the previous value. But the "Reset" button uses the normal form, because it always sets the count back to 0.

> Note
>
> Unlike the `setState` method found in class components, `useState` does not automatically merge update objects. You can replicate this behavior by combining the function updater form with object spread syntax:
>
> ```js
> setState(prevState => {
>   // Object.assign would also work
>   return {...prevState, ...updatedValues};
> });
> ```
>
> Another option is `useReducer`, which is more suited for managing state objects that contain multiple sub-values.

#### Lazy initialization

The `initialState` argument is the state used during the initial render. In subsequent renders, it is disregarded. If the initial state is the result of an expensive computation, you may provide a function instead, which will be executed only on the initial render:

```js
const [state, setState] = useState(() => {
  const initialState = someExpensiveComputation(props);
  return initialState;
});
```

### `useEffect`

```js
useEffect(didUpdate);
```

Accepts a function that contains imperative, possibly effectful code.

Mutations, subscriptions, timers, logging, and other side-effects are not allowed inside the main body of a function component (referred to as React's _render phase_). Doing so will lead to confusing bugs and inconsistencies in the UI.

Instead, use `useEffect`. The function passed to `useEffect` will run after the render is committed to the screen. Think of effects as an escape hatch from React's purely functional world into the imperative world.

By default, effects run after every completed render, but you can choose to fire it [only when certain values have changed](#conditionally-firing-an).

#### Cleaning up an effect

Often, effects create resources that need to be cleaned up before the component leaves the screen, such as a subscription or timer ID. To do this, the function passed to `useEffect` may return a clean-up function. For example, to create a subscription

```js
useEffect(() => {
  const subscription = props.source.subscribe();
  return () => {
    // Clean up the subscription
    subscription.unsubscribe();
  };
});
```

The clean-up function runs before the component is removed from the UI to prevent memory leaks. Additionally, if a component renders multiple times (as they typically do), the **previous effect is cleaned up before executing the next effect**. In our example, this means a new subscription is created on every update. To avoid firing an effect on every update, refer to the next section.

#### Timing of effects

Unlike `componentDidMount` and `componentDidUpdate`, the function passed to `useEffect` fires **after** layout and paint, during a deferred event. This makes it suitable for the many common side-effects, like setting up subscriptions and event handlers, because that type of work shouldn't block the browser from updating the screen.

However, not all effects can be deferred. Specifically, any effect that mutates the DOM must fire synchronously before the next paint so that the user does not perceive a visual inconsistency. (The distinction is conceptually similar to passive versus active event listeners.) For these types of effects, provides two additional hooks: [`useMutationEffect`](#usemutationeffect) and [`useLayoutEffect`](#uselayouteffect). These hooks have the same signature as `useEffect`, and only differ in when they are fired.

Although `useEffect` is deferred until after the browser has painted, it's guaranteed to fire before a subsequent mutation. React will always flush a previous render's effects before committing new ones.

#### Conditionally firing an effect

The default behavior for effects is to fire the effect after every completed render. That way an effect is always re-created if one of its inputs changes.

However, this may be overkill in some cases, like the subscription example from the previous section. We don't need to create a new subscription on every update, only if the `source` props has changed.

To implement this, pass a second argument to `useEffect` that is the array of values that the effect depends on. Our updated example now looks like this:

```js
useEffect(
  () => {
    const subscription = props.source.subscribe();
    return () => {
      subscription.unsubscribe();
    };
  },
  [props.source],
);
```

Now the subscription will only be re-created when `props.source` changes.

> Note
>
> The array of inputs is not passed as arguments to the effect function. Conceptually, though, that's what they represent: every value referenced inside the effect function should also appear in the inputs array. In the future, a sufficiently advanced compiler could create this array automatically.

### `useContext`

```js
const context = useContext(Context);
```

Accepts a context object (the value returned from `React.createContext`) and returns the current context value, as given by the nearest context provider for the given context.

## Additional Hooks

The following hooks are either variants of the basic ones from the previous section, or only needed for specific edge cases. Don't stress about learning them up front.

### `useReducer`

```js
const [state, dispatch] = useReducer(reducer, initialState);
```

An alternative to [`useState`](https://our.intern.facebook.com/intern/wiki/React_Hooks/#usestate). Accepts a reducer of type `(state, action) => newState`, and returns the current state paired with a `dispatch` method. (If you're familiar with Redux, you already know how this works.)

Here's the counter example from the [`useState`](https://our.intern.facebook.com/intern/wiki/React_Hooks/#usestate) section, rewritten to use a reducer:

```js
const RESET = 'RESET';
const INCREMENT = 'INCREMENT';
const DECREMENT = 'DECREMENT';

const initialState = {count: 0};

function reducer(state, action) {
  switch (action.type) {
    case RESET:
      return initialState;
    case INCREMENT:
      return {count: state.count + 1};
    case DECREMENT:
      return {count: state.count - 1};
  }
}

function Counter({initialCount}) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({type: RESET})}>
        Reset
      </button>
      <button onClick={() => dispatch({type: INCREMENT})}>+</button>
      <button onClick={() => dispatch({type: DECREMENT})}>-</button>
    </>
  );
}
```

#### Lazy initialization

`useReducer` accepts an optional third argument, `initialAction`. If provided, the initial action is applied during the initial render. This is useful for computing an initial state that includes values passed via props:

```js
const RESET = 'RESET';
const INCREMENT = 'INCREMENT';
const DECREMENT = 'DECREMENT';

const initialState = {count: 0};

function reducer(state, action) {
  switch (action.type) {
    case RESET:
      return {count: action.payload};
    case INCREMENT:
      return {count: state.count + 1};
    case DECREMENT:
      return {count: state.count - 1};
  }
}

function Counter({initialCount}) {
  const [state, dispatch] = useReducer(
    reducer,
    initialState,
    {type: RESET, payload: props.initialCount},
  );

  return (
    <>
      Count: {state.count}
      <button
        onClick={() => dispatch({type: RESET, payload: props.initialCount})}>
        Reset
      </button>
      <button onClick={() => dispatch({type: INCREMENT})}>+</button>
      <button onClick={() => dispatch({type: DECREMENT})}>-</button>
    </>
  );
}
```

`useReducer` is usually preferable to `useState` when you have complex state logic that involves multiple sub-values. It also lets you optimize performance for components that trigger deep updates because [you can pass `dispatch` down instead of callbacks](/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down).

### `useCallback`

```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```

Returns a memoized callback.

Pass an inline callback and an array of inputs. `useCallback` will return a memoized version of the callback that only changes if one of the inputs has changed. This is useful when passing callbacks to optimized child components that rely on reference equality to prevent unnecessary renders (e.g. `shouldComponentUpdate`).

> Note
>
> The array of inputs is not passed as arguments to the callback. Conceptually, though, that's what they represent: every value referenced inside the callback should also appear in the inputs array. In the future, a sufficiently advanced compiler could create this array automatically.

### `useMemo`

```js
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

Returns a memoized value.

Pass a "create" function and an array of inputs. `useMemo` will only recompute the memoized value when one of the inputs has changed. This optimization helps to avoid expensive calculations on every render.

> Note
>
> The array of inputs is not passed as arguments to the function. Conceptually, though, that's what they represent: every value referenced inside the function should also appear in the inputs array. In the future, a sufficiently advanced compiler could create this array automatically.

### `useRef`

```js
const refContainer = useRef(initialValue);
```

`useRef` returns a [ref object](/link/goes/here) whose `current` pointer is initialized to the passed argument (`initialValue`). The typical use case is to obtain a reference to a child component:

```js
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` points to the mounted text input element
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```

Note that `useRef()` is useful for more than DOM elements. It's [handy for keeping any mutable value around](/docs/hooks-faq.html#is-there-something-like-instance-variables), and is similar to how you'd use instance fields in classes.

### `useAPI`

TODO

### `useMutationEffect`

The signature is identical to `useEffect`, but fires synchronously during the same phase that React performs its DOM mutations. Use this to perform custom DOM mutations.

See the section on [effect timing](#timing-of-effects) for a fuller explanation.

### `useLayoutEffect`

The signature is identical to `useEffect`, but fires synchronously *after* all DOM mutations. Use this to read layout from the DOM and synchronously re-render. Updates scheduled inside `useLayoutEffect` will be flushed synchronously, before the browser has a chance to paint.

> Tip
>
> If you're migrating code from a class component, `useLayoutEffect` fires in the same phase as `componentDidMount` and `componentDidUpdate`, so if you're unsure of which effect hook to use, it's probably the least risky.

See the section on [effect timing](#timing-of-effects) for a fuller explanation.
