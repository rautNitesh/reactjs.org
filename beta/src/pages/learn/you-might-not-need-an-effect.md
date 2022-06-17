---
title: 'You Might Not Need an Effect'
---

<Intro>

Effects are an escape hatch from the React paradigm. They let you "step outside" of React and synchronize your components with some external system like a non-React widget, network, or the browser DOM. If there is no external system involved (for example, if you want to update a component's state when some props or state change), you shouldn't need an Effect. Removing unnecessary Effects will make your code easier to follow, faster to run, and less error-prone.

</Intro>

<YouWillLearn>

* What to keep in mind when writing Effects
* How to remove unnecessary Effects from your components

</YouWillLearn>

## Principles for writing Effects {/*principles-for-writing-effects*/}

Similar to [principles for structuring state](/learn/choosing-the-state-structure#principles-for-structuring-state), there are a few principles that can help you with writing Effects:

* **Avoid cascading updates.** When you update your component's state, React will first call your component functions to calculate what should be on the screen. Then React will ["commit"](/learn/render-and-commit) these changes to the DOM, updating the screen. Then React will run your Effects. If your Effect *also* immediately updates the state, this restarts the whole process from scratch! React has to call your component functions again, and then update the DOM again. This is called a *cascading update*. Some cascading updates are unavoidable (for example, you might need to render a tooltip and measure its content size before you can decide how to position it). But in general, you should avoid cascading updates, and try to respond to every interaction with a single render pass.

* **Keep interaction-specific logic in the event handlers.** Every interaction begins with an event. In the event handler, you have the most information about what happened: for example, you know which concrete button was pressed. If you update the state and then put some logic into the Effect, then by the time the Effect runs, you don't know *which* particular interaction has happened. Effects "lose" information about the cause of interaction, so they work best to [synchronize external systems with React:](/learn/synchronizing-with-effects#what-are-effects-and-how-are-they-different-from-events) for example, to keep a non-React widget in sync with the React state, or to keep the React state in sync with information from the network. In those cases, the cause of the interaction does not matter: you want to do something *as a result of rendering*. For specific interactions, like clicking a Buy button, it makes sense to put most logic into the event handler.

* **Keep the data flowing down.** In React, data flows from the parent components to their children. When you see something wrong on the screen, you can trace where the information comes from by going up the component chain until you find which component passes the wrong prop or has the wrong state. But if you have child components with Effects that update the state of their parent components, the data flow becomes more difficult to trace. When possible, avoid updating the parent component state from a child component's Effect. Instead, see if there's an appropriate event that the parent component can listen to.

These principles might feel a bit abstract at first. The examples below will help you develop the right intuition!

## How to remove unnecessary Effects {/*how-to-remove-unnecessary-effects*/}

To put these principles in practice, you will usually follow three steps:

1. Calculate as much as you can during rendering.
2. Move all the interaction-specific logic into the event handlers.
3. Keep the remaining synchronization logic in Effects, if there is any left.

Let's look at the most common patterns, and how you can simplify them.

### Updating the state based on other state {/*updating-the-state-based-on-other-state*/}

Suppose you have a component with two state variables: `firstName` and `lastName`. You want to calculate a `fullName` from them by concatenating them. Moreover, you'd like `fullName` to update whenever `firstName` or `lastName` change. Your first instinct might be to add a `fullName` state variable and update it in an effect:

```js {5-9}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  // 🔴 Bad: redundant state and unnecessary Effect
  const [fullName, setFullName] = useState('');
  useEffect(() => {
    setFullName(firstName + ' ' + lastName);
  }, [firstName, lastName]);
```

This is more complicated than necessary. It is inefficient too: it does an entire render pass with a stale value for `fullName`, then immediately re-renders with the updated value. Remove both the state variable and the Effect:

```js {5-6}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  // ✅ Good: calculated during rendering
  const fullName = firstName + ' ' + lastName;
```

**When something can be calculated from the existing props or state, [don't put it in state.](/learn/choosing-the-state-structure#avoid-redundant-state) Instead, calculate it during rendering.** This makes your code faster (you avoid the extra "cascading" updates), simpler (you remove some code), and less error-prone (you avoid bugs caused by different state variables getting out of sync with each other). If this approach feels new to you, [Thinking in React](/learn/thinking-in-react#step-3-find-the-minimal-but-complete-representation-of-ui-state) has some guidance on what should go into state.

### Caching expensive calculations {/*caching-expensive-calculations*/}

This component computes `visibleTodos` by taking the `todos` it receives by props and filtering them according to the `filter` prop. You might feel tempted to store the result in a state variable and update it in an Effect:

```js {4-8}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // 🔴 Bad: redundant state and unnecessary Effect
  const [visibleTodos, setVisibleTodos] = useState([]);
  useEffect(() => {
    setVisibleTodos(getFilteredTodos(todos, filter));
  }, [todos, filter]);
```

Like in the earlier example, this is both unnecessary and inefficient. First, remove the state and the Effect:

```js {4-5}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // ✅ This is fine if getFilteredTodos() is not slow.
  const visibleTodos = getFilteredTodos(todos, filter);
```

In many cases, this code is fine! But maybe `getFilteredTodos()` is slow or you have a lot of `todos`. In that case you don't want to recalculate `getFilteredTodos()` if some unrelated state variable like `newTodo` has changed.

You can cache (or ["memoize"](https://en.wikipedia.org/wiki/Memoization)) an expensive calculation by wrapping it [`useMemo`](/apis/usememo) Hook:

```js {6-9}
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  const visibleTodos = useMemo(() => {
    // ✅ Does not re-run unless todos or filter change
    return getFilteredTodos(todos, filter);
  }, [todos, filter]);
```

Or, written as a single line:

```js {6-7}
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // ✅ Does not re-run getFilteredTodos() unless todos or filter change
  const visibleTodos = useMemo(() => getFilteredTodos(todos, filter), [todos, filter]);
```

**This tells React that you don't want the inner function to re-run unless either `todos` or `filter` have changed.** React will remember the return value of `getFilteredTodos()` during the initial render. During the next renders, it will check if `todos` or `filter` are different. If they're the same as last time, `useMemo` will return the last result it has stored. But if they are different, React will call the wrapped function again (and store _that_ result instead).

The function you wrap in [`useMemo`](/apis/usememo) runs during rendering, so this only works for [pure calculations](/learn/keeping-components-pure).

<DeepDive title="How to tell if a calculation is expensive?">

In general, unless you're creating or looping over thousands of objects, it's probably not expensive. If you want to get more confidence, you can add a console log to measure the time spent in a piece of code:

```js {1,3}
console.time('filter array');
const visibleTodos = getFilteredTodos(todos, filter);
console.timeEnd('filter array');
```

Perform the interaction you're measuring (for example, typing into the input). You will then see logs like `filter array: 0.15ms` in your console. If the overall logged time adds up to a significant amount (say, `5ms` or more), it might make sense to memoize that calculation. As an experiment, you can then wrap the calculation in `useMemo` to verify whether the total logged time has significantly decreased:

```js
console.time('filter array');
const visibleTodos = useMemo(() => {
  return getFilteredTodos(todos, filter); // Skipped if todos and filter haven't changed
}, [todos, filter]);
console.timeEnd('filter array');
```

Keep in mind that your machine is probably faster than your users' so it's a good idea to test the performance with an artificial slowdown. For example, Chrome offers a [CPU Throttling](https://developer.chrome.com/blog/new-in-devtools-61/#throttling) option for this.

Also note that measuring performance in development will not give you the most accurate results. (For example, when [Strict Mode](/apis/strictmode) is on, you will see each component render twice rather than once.) To get the most accurate timings, build your app for production and test it on a device like your users have.

</DeepDive>

### Resetting the state on a prop change {/*resetting-the-state-on-a-prop-change*/}

TODO

### Moar sections {/*moar-sections*/}

TODO

