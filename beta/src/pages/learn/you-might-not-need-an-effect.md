---
title: 'You Might Not Need an Effect'
---

<Intro>

Effects are an escape hatch from the React paradigm. They let you "step outside" of React and synchronize your components with some external system like a non-React widget, network, or the browser DOM. If there is no external system involved (for example, if you want to update a component's state when some props or state change), you shouldn't need an Effect. Removing unnecessary Effects will make your code easier to follow, faster to run, and less error-prone.

</Intro>

<YouWillLearn>

* How to remove unnecessary Effects from your components

</YouWillLearn>

## Updating state based on other state {/*updating-state-based-on-other-state*/}

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

## Caching expensive calculations {/*caching-expensive-calculations*/}

This component computes `visibleTodos` by taking `allTodos` it receives by props and filtering them according to the currently chosen `filter` prop. You might feel tempted to update a state variable in an Effect:

```js {4-8}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // 🔴 Bad: redundant state and unnecessary Effect
  const [visibleTodos, setVisibleTodos] = useState([]);
  useEffect(() => {
    setVisibleTodos(getFilteredTodos(todos, filter));
  }, [todos, filter]);
```

First, remove the unnecessary state and Effect:

```js {4-5}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // 🟡 This is fine if getFilteredTodos() is not slow.
  const visibleTodos = getFilteredTodos(todos, filter);
```

In many cases, this code is fine! But maybe `getFilteredTodos()` is slow or you have a lot of `todos`. In that case you don't want to recalculate `getFilteredTodos()` if some unrelated state variable like `newTodo` has changed.

You can cache (or ["memoize"](https://en.wikipedia.org/wiki/Memoization)) an expensive calculation by wrapping it [`useMemo`](/apis/usememo) Hook:

```js {6-9}
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // ✅ Does not re-run getFilteredTodos() unless todos or filter change
  const visibleTodos = useMemo(() => {
    return getFilteredTodos(todos, filter);
  }, [todos, filter]);
```

**This tells React that you don't want the inner function to re-run unless either `todos` or `filter` have changed.** React will remember the return value of `getFilteredTodos()` during the initial render. During the next renders, it will check if `todos` or `filter` are different. If they're the same as last time, `useMemo` will return the last result it has stored. But if one of them changes, React will call the wrapped function again (and store _that_ result instead).

In other words, [`useMemo`](/apis/usememo) lets you skip re-running a calculation during re-renders if its inputs have not changed. Keep in mind that the function you wrap in [`useMemo`](/apis/usememo) runs during rendering, so it should be a [pure calculation](/learn/keeping-components-pure).

## TODO: Moar sections {/*todo-moar-sections*/}

TODO


