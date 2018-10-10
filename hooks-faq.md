---
id: hooks-faq
title: Hooks FAQ
permalink: docs/hooks-faq.html
prev: hooks-reference.html
---

This page answers some of the commonly asked questions about [Hooks](/docs/hooks-overview.html).

- [Adoption Strategy](#adoption-strategy)
- [From Classes to Hooks](#from-classes-to-hooks)
- [Performance Optimizations](#performance-optimizations)

## Adoption Strategy

### Do I need to rewrite all my class components?

No. There are [no plans](/docs/hooks-intro.html#gradual-adoption-strategy) to remove classes from React -- we all need to keep shipping products and can't afford rewrites. We recommend to start trying Hooks in new code.

### How much of my React knowledge stays relevant?

Hooks are a more direct way to use the React features you already know -- such as state, lifecycle, context, and refs. They don't fundamentally change how React works, and your knowledge of components, props, and top-down data flow is just as relevant.

Hooks do have a learning curve of their own. If there's something missing in this documentation, [raise an issue](https://github.com/reactjs/reactjs.org/issues/new) and we'll try to help.

### Can I mix classes and Hooks?

You can't use Hooks *inside* of a class component, but you can definitely mix classes and function components with Hooks in a single tree. Whether a component is a class or a function that uses Hooks is an implementation detail of that component.

### Should I use Hooks or classes in the future?

In the short term, we suggest to start trying Hooks in some of the newer components you write. Make sure everyone on your team is onboard with using them and familiar with this documentation. We don't recommend rewriting your existing classes to Hooks unless you planned to rewrite them anyway (e.g. to fix bugs). In the longer term, we expect Hooks to be the primary way people write React components.

### Do Hooks cover all use cases for classes?

Our goal is for Hooks to cover all use cases for classes as soon as possible. It is a very early time for Hooks, so some integrations like DevTools support or Flow/TypeScript typings aren't ready yet. Some third-party libraries might also not be compatible with Hooks at the moment. There are no Hook equivalents to the uncommon `getSnapshotBeforeUpdate` and `componentDidCatch` lifecycles yet, but we plan to add them soon.

### Do Hooks replace render props and higher-order components?

Often, render props and higher-order components render only a single child. We think Hooks serve this use case more ergonomically. There is still a place for both patterns (for example, a virtual scroller component might have a `renderItem` prop, or a visual container component might have its own DOM structure). But we expect that in most cases Hooks will be sufficient and help reduce nesting in your tree.

### Do Hooks work with static typing?

Hooks were designed with static typing in mind. Because they're functions, they are easier to type correctly than e.g. higher-order components. We plan to include Flow definitions for React Hooks in the future. While we don't maintain TypeScript definitions ourselves, we'd be happy to give feedback on the community typing proposals.

Importantly, custom Hooks give you the power to constrain React API if you'd like to type them more strictly in some way. React gives you the primitives, but you can combine them in different ways than what we provide out of the box.

### What exactly do the lint rules enforce?

We provide a linter plugin that enforces [rules of Hooks](/docs/hooks-rules.html) to avoid bugs. It assumes that any function starting with "`use`" and a capital letter right after it is a Hook. We recognize it's a compromise and there will likely be false positives, but without an ecosystem-wide convention there is just no way to make Hooks work well -- and longer names will discourage people from either adopting Hooks or following the convention.

In particular, the rule enforces that:

* Calls to Hooks are either inside a `PascalCase` function (assumed to be a component) or another `useSomething` function (assumed to be a custom Hook).
* Hooks are called in the same order on every render.

There are a few more heuristics, and they might change over time as we fine-tune the rule to balance finding bugs with avoiding false positives.

## From Classes to Hooks

### What does `const [thing, setThing] = useState()` mean?

If you're not familiar with the array destructuring syntax, check out the [explanation](/docs/hooks-state.html#tip-what-do-square-brackets-mean) in the State Hook documentation.

### How do lifecycle methods correspond to Hooks?

* `constructor`: Function components don't need a constructor. You can initialize the state in the [`useState`](/docs/hooks-reference.html#usestate) call. If computing it is expensive, you can pass a function to `useState`.

* `getDerivedStateFromProps`: Schedule an update [while rendering](#how-do-i-implement-getderivedstatefromprops) instead.

* `shouldComponentUpdate`: See `React.pure` [below](#how-do-i-implement-getderivedstatefromprops).

* `render`: This is the function component body itself.

* `componentDidMount`, `componentDidUpdate`, `componentWillUnmount`: The [`useEffect` Hook](/docs/hooks-reference.html#useeffect) can express all combinations of these (including [less](#can-i-skip-an-effect-on-updates) [common](#can-i-run-an-effect-only-on-updates) cases).

* `componentDidCatch` and `getDerivedStateFromError`: There are no Hook equivalents for these methods yet, but they will be added soon.

### How do I implement `getDerivedStateFromProps`?

While you probably [don't need it](/blog/2018/06/07/you-probably-dont-need-derived-state.html), for the rare cases that you do (such as implementing a `<Transition>` component), you can update the state right during rendering. React will re-run the component with updated state immediately after exiting the first render so it wouldn't be expensive.

```js
function ScrollView({row: newRow}) {
  let [isScrollingDown, setIsScrollingDown] = useState(false);
  let [row, setRow] = useState(null);

  if (row !== newRow) {
    // Row changed since last render. Update isScrollingDown.
    setIsScrollingDown(row !== null && newRow > row);
    setRow(newRow);
  }

  return `Scrolling down: ${isScrollingDown}`;
}
```

This might look strange at first, but an update during rendering is exactly what `getDerivedStateFromProps` has always been like conceptually.

### Is there something like instance variables?

Yes! The [`useRef()`](/docs/hooks-reference.html#useref) Hook isn't just for DOM refs. The "ref" object is a generic container whose `current` property hold any mutable value, similar to an instance property on a class.

You can write to it from inside `useEffect`:

```js{2,8,15}
function Timer() {
  const intervalRef = useRef();

  useEffect(() => {
    const id = setInterval(() => {
      // ...
    });
    intervalRef.current = id;
    return () => {
      clearInterval(intervalRef.current);
    };
  });

  // ...
}
```

If we just wanted to set an interval, we wouldn't need this (`id` could be local to the effect), but it's useful if we want to clear the interval from an event handler:

```js{3}
  // ...
  function handleCancelClick() {
    clearInterval(intervalRef.current);
  }
  // ...
```

Conceptually, you can think of refs as similar to instance variables in a class. Avoid setting refs during rendering -- this can lead to surprising behavior.

### Can I run an effect only on updates?

Yes, but you'd need to keep track of whether this is a mount or an update yourself. For example, you can use a ref ([as described here](#is-there-something-like-instance-variables)) to keep track of whether you have already mounted:

```js{2,8-9}
function AlertOnUpdates() {
  const hasMountedRef = useRef();

  useEffect(() => {
    if (hasMountedRef.current) {
      alert('this is an update');
    } else {
      // Set a flag for next time effect runs
      hasMountedRef.current = true;
    }
  });

  // ...
}
```

If you find yourself doing this often, you can create a custom Hook for this:

```js{2,8}
function AlertOnUpdates() { 
  useUpdateEffect(() => {
    alert('this is an update');
  });
  // ...
}

function useUpdateEffect(fn) {
  const hasMountedRef = useRef();
  useEffect(() => {
    if (hasMountedRef.current) {
      fn();
    } else {
      hasMountedRef.current = true;
    }
  });
}
```

You can also do the opposite and [skip the effect on updates](#can-i-skip-an-effect-on-updates).

### How to get the previous props or state?

Currently, you can do it manually [with a ref](#is-there-something-like-instance-variables):

```js{6,8}
function Counter() {
  const [count, setCount] = useState(0);

  const prevCountRef = useRef();
  useEffect(() => {
    prevCountRef.current = count;
  });
  const prevCount = prevCountRef.current;

  return <h1>Now: {count}, before: {prevCount}</h1>;
}
```

This might be a bit convoluted but you can extract it into a custom Hook:

```js{3,7}
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  return <h1>Now: {count}, before: {prevCount}</h1>;
}

function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

Note how this would work for props, state, or any other calculated value.

```js{5}
function Counter() {
  const [count, setCount] = useState(0);

  const calculation = count * 100;
  const prevCalculation = usePrevious(calculation);
  // ...
  ```

It's possible that in the future React will provide a `usePrevious` Hook out of the box since it's a relatively common use case.

### Why use `useContext` when I can use <Context.Consumer>?

They behave the same way (pulling the context value from the nearest matching context provider), but with `useContext` you can use values from Context within your render and other hooks in one line of code (no render props) and avoiding "wrapper hell".

## Performance Optimizations

### Can I skip an effect on updates?

You can completely skip an effect on updates by specifying `[]` as the second argument to `useEffect`. See [conditionally firing an effect](/docs/hooks-reference.html#conditionally-firing-an-effect). Note that forgetting to handle updates often [introduces bugs](/docs/hooks-effect.html#explanation-why-next-effects-replace-the-previous-ones), which is why this isn't the default behavior.

### How do I implement `shouldComponentUpdate`?

You can wrap a function component with `React.pure` to shallowly compare its props:

```js
const Button = React.pure((props) => {
  // your component
});
```

It's not a Hook because it doesn't compose like Hooks do. `React.pure` is equivalent to `PureComponent`, but it only compares props. (You can also add a second argument to specify a custom comparison function.)

`React.pure` doesn't compare state because there is no single state object to compare. But you can make children pure too, or even [optimize individual children](/docs/hooks-faq.html#how-to-memoize-calculations).


### How to memoize calculations?

The [`useMemo`](/docs/hooks-reference.html#usememo) Hook lets you do this:

```js
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

Conveniently, it also lets you skip an expensive re-render of a child:

```js
function Parent({ a, b }) {
  // Only re-rendered if `a` changes:
  const child1 = useMemo(() => <Child a={a} />, [a]);
  // Only re-rendered if `b` changes:
  const child2 = useMemo(() => <Child b={b} />, [b]);
  return (
    <>
      {child1}
      {child2}
    </>
  )
}
```

Note that this approach won't work in a loop because Hook calls [can't](/docs/hooks-rules.html) be placed inside loops. But you can extract a separate component for the list item, and call `useMemo` there.

### Isn't creating functions in render slow?

For the initial render performance or server rendering (as well as updates that include an initial render of complex trees), Hooks don't create some of the overhead that is necessary for classes. Creating class instances and defining many event handlers in the constructor that are bound to the instances isn't free either, and Hooks don't need that.

The raw performance of closures compared to classes depends on the browser, but it doesn't differ significantly in modern browsers except for extreme scenarios. And if it's the function creation itself is the bottleneck, you likely have a lower hanging fruit to optimize, such as skipping re-rendering or introducing virtualization.

Keep in mind that **idiomatic code using Hooks doesn't need the deep component tree nesting** that is prevalent in the codebases that use higher-order components, render props, and context. So while there are more function allocations, there are fewer element allocations and less reconciliation overhead. Hooks also [make memoization easier](/docs/hooks-faq.html#how-to-memoize-calculations) and more targeted. You can optimize individual parts of the tree by memoizing them right in the parent component.

Most of performance concerns around inline functions in React are related to how passing different callbacks down on each render breaks `shouldComponentUpdate` optimizations in child components. Hooks approach this problem from two sides.

The [`useCallback`](/docs/hooks-reference.html#usecallback) Hook lets you keep the same callback reference between re-renders without adding a single extra line of code:

```js{2}
// Will not change unless `a` or `b` changes
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

More importantly, with Hooks you can stop passing callbacks deeply, as explained below.

### How to avoid passing callbacks down?

Instead of trying to optimize callbacks, we recommend to use callback props only for components that are relatively close to the DOM event handlers, rather than for the state update logic of your entire application.

We recommend managing the state of complex components higher up in the tree with [`useReducer`](/docs/hooks-reference.html#usereducer) and passing `dispatch` to child components through context:

```js{4,5}
const TodosDispatch = React.createContext(null);

function Todos() {
  // Tip: `dispatch` doesn't change between re-renders
  const [todos, dispatch] = useReducer(todosReducer);

  return (
    <TodosDispatch.Provider value={dispatch}>
      <DeepTree todos={todos} />
    </TodosDispatch.Provider>
  );
}
```

This `dispatch` reference wouldn't change between re-renders. Any child in the tree inside `Todos` can read it and call it to pass actions up to `Todos`:

```js{2,3}
const DeepChild = React.pure(props => {
  // If we want to perform an action, we can get dispatch from context.
  const dispatch = useContext(TodosDispatch);

  function handleClick() {
    dispatch({ type: 'add', text: 'hello' });
  }

  return (
    <button onClick={handleClick}>Add todo</button>
  );
});
```

This is both more convenient from the maintenance perspective (no need to keep forwarding callbacks), and avoids the callback problem altogether. Passing `dispatch` down like this is the recommended pattern for deep updates.

Note that you can still choose whether to pass the *state* down as props (more explicit) or as context (more convenient for very deep updates). Keeping state and dispatch context separated is a useful optimization because the `dispatch` context never changes.

### How to read an often-changing value from `useCallback`?

>Note
>
>We [recommend to **pass `dispatch` down in context** rather than individual callbacks in props](#how-to-avoid-passing-callbacks-down). The approach below is only mentioned here for completeness and as an escape hatch.

In some rare cases you might need to memoize a callback with [`useCallback`](/docs/hooks-reference.html#usecallback) but the memoization doesn't work very well because the inner function has to be re-created too often. If the function you're memoizing is an event handler and isn't used during rendering, you can use [ref as an instance variable](#is-there-something-like-instance-variables), and save the last committed value into it manually:

```js{6,10}
function Form() {
  const [text, updateText] = useState('');
  const textRef = useRef();

  useEffect(() => {
    textRef.current = text; // Write it to the ref
  });

  const handleSubmit = useCallback(() => {
    const currentText = textRef.current; // Read it from the ref
    alert(currentText);
  }, [textRef]); // Don't recreate handleSubmit like [text] would do

  return (
    <>
      <input value={text} onChange={e => updateText(e.target.value)} />
      <ExpensiveTree onSubmit={handleSubmit} />
    </>
  );
}
```

This is a rather convoluted pattern but it shows that you can do this escape hatch optimization if you need it. It's more bearable if you extract it to a custom Hook:

```js{4,16}
function Form() {
  const [text, updateText] = useState('');
  // Will be memoized even if `text` changes:
  const handleSubmit = useEventCallback(() => {
    alert(text);
  }, [text]);

  return (
    <>
      <input value={text} onChange={e => updateText(e.target.value)} />
      <ExpensiveTree onSubmit={handleSubmit} />
    </>
  );
}

function useEventCallback(fn, dependencies) {
  const ref = useRef(() => {
    throw new Error('Cannot call an event handler while rendering.');
  });

  useEffect(() => {
    ref.current = fn;
  }, [fn, ...dependencies]);

  return useCallback(() => {
    const fn = ref.current;
    return fn();
  }, [ref]);
}
```

In either case, we **don't recommend this pattern** and only show it here for completeness. Instead, it is preferable to [avoid passing callbacks deep down](#how-to-avoid-passing-callbacks-down).

## Under the Hood

### How does React associate Hook calls with components?

React keeps track of the currently rendering component. Thanks to the [Rules of Hooks](/docs/hooks-rules.html), we know that Hooks are only called from React components (or custom Hooks -- which are also only called from React components).

There is an internal list of "memory cells" associated with each component. They're just JavaScript objects where we can put some data. When you call a Hook like `useState()`, it reads the current cell (or initializes it during the first render), and then moves the pointer to the next one. This is how multiple `useState()` calls each get independent local state.

### What is the prior art for Hooks?

Hooks synthesize ideas from several different sources:

* Our old experiments with functional APIs in the [react-future](https://github.com/reactjs/react-future/tree/master/07%20-%20Returning%20State) repository.
* React community's experiments with render prop APIs, including [Ryan Florence](https://github.com/ryanflorence)'s [React Component Component](https://github.com/reactions/component).
* [Dominic Gannaway](https://github.com/trueadm)'s [`adopt` keyword](https://gist.github.com/trueadm/17beb64288e30192f3aa29cad0218067) proposal as a sugar syntax for render props.
* State variables and state cells in [DisplayScript](http://displayscript.org/introduction.html).
* [Reducer components](https://reasonml.github.io/reason-react/docs/en/state-actions-reducer.html) in ReasonReact.
* [Subscriptions](http://reactivex.io/rxjs/class/es6/Subscription.js~Subscription.html) in Rx.
* [Algrebraic effects](https://github.com/ocamllabs/ocaml-effects-tutorial#2-effectful-computations-in-a-pure-setting) in Multicore OCaml.

[Sebastian Markbåge](https://github.com/sebmarkbage) came up with the original design for Hooks, later refined by [Andrew Clark](https://github.com/acdlite), [Sophie Alpert](https://github.com/sophiebits), [Dominic Gannaway](https://github.com/trueadm), and other members of the React team.
