---
title: "Intro to React Hooks"
description: "A quick intro to some basic React Hooks APIs"
tags: [ "Javascript", "react", "hooks" ]
categories: [ "Development", "Javascript", "How To" ]
date: 2019-05-11T10:54:00-04:00
---

With the release of React 16.8, React Hooks are now available in a stable
release!

### What are React Hooks?
Hooks allow you to use state and other React features without writing a class.

---

### What are some of the Basic Hooks Available?
[useState](https://reactjs.org/docs/hooks-reference.html#usestate)

a hook to allow a function component to utilize state

[useReducer](https://reactjs.org/docs/hooks-reference.html#usereducer)

an alternative to `useState`. Accepts a reducer of type `(state, action) => newState`,
and returns the current state paired with a dispatch method. (If you’re familiar
with Redux, you already know how this works.)

[useEffect](https://reactjs.org/docs/hooks-reference.html#useeffect)

a hook to allow you to declare a function that contains imperative, possibly
effectful code. Mutations, subscriptions, timers, logging, and other side
effects are not allowed inside a function component. Doing so leads to confusing
bugs and inconsistencies in the UI.

The function passed to useEffect will run after the render is committed to the
screen. By default, effects run after every completed render.
          
[useContext](https://reactjs.org/docs/hooks-reference.html#usecontext)

makes it easy to use React context without using a class component. React
context gives the ability to easily share state between multiple components,
similar to Redux.

---

### Show Me Some Code!

To help showcase how/when to use these basic hooks I will show an example of
a class component doing a "thing/feature" and then show how to achieve the same
"thing/feature" using a new React hook.

#### useState
For the useState example I will use the classic counter example to keep it simple.

**Class Component Solution**
<iframe src="https://codesandbox.io/embed/ol4v9360yz?fontsize=14&view=editor" title="Class Component with State" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

**Function Component Solution**
<iframe src="https://codesandbox.io/embed/7j165nl270?fontsize=14&view=editor" title="Function Component with useState" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

The function component contains far less required ceremony to utilize a simple
counter state. Less things that you need to remember to do and wire.

#### useReducer
For the useReducer example I will repeat the counter example from before with
some increased options

**Class Component Solution (with the help of Redux)**
<iframe src="https://codesandbox.io/embed/znj86075x4?fontsize=14&view=editor" title="Class Component with Redux Counter" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

**Function Component Solution**
<iframe src="https://codesandbox.io/embed/zl6qll6z9l?fontsize=14&view=editor" title="Function Component with Reducer" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

The function component contains far less ceremony to setup to utilize a simple
counter reducer. Less things that you need to remember to do and wire. However
this isn't a perfect 1:1 comparison as Redux is meant to be global state whereas
`useReducer` is meant to be local state.


#### useEffect
For the useEffect example I will use a corporate buzzword API to load a random
corporate buzzword into the component.

**Class Component Solution**
<iframe src="https://codesandbox.io/embed/qzw6z38rvj?fontsize=14&view=editor" title="Class Component with Side Effect" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

**Function Component Solution**
<iframe src="https://codesandbox.io/embed/8zvoow18ml?fontsize=14&view=editor" title="Function Component with useEffect" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

In the function component example we get the effect to only run once by
specifying an empty array as the second argument, [reference to how this works](https://reactjs.org/docs/hooks-reference.html#conditionally-firing-an-effect)

### Use Context
For the useContext example I will show how you can use a shared theme data
across components.

**Class Component Solution**
<iframe src="https://codesandbox.io/embed/mz057mj45x?fontsize=14&view=editor" title="Class Component with Context" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe> 

**Function Component Solution**
<iframe src="https://codesandbox.io/embed/x9wj4yl2op?fontsize=14&view=editor" title="Function Component with Context" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

These 2 examples are almost identical, but in my opinion I think the function
component example is a bit easier to use.

And with that we have come to the conclusion of some very quick and dirty diffs
of class components vs function components with new React Hooks. I hope these
examples can help you internalize how and when you can use some of the available
React Hooks.

Here are some great additional resources you can dive into to learn more about
React Hooks in-depth.

---

### Dan Abramov's React Conf 2018 Hooks Talk 
<iframe width="560" height="315" src="https://www.youtube.com/embed/dpw9EHDh2bM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>  

---

### Links to Other Resources
[Hooks API Reference](https://reactjs.org/docs/hooks-reference.html)

[Hooks General Overview](https://reactjs.org/docs/hooks-overview.html)

[Introducing Hooks](https://reactjs.org/docs/hooks-intro.html) - explains why hooks were added to React

[Hooks at a Glance](https://reactjs.org/docs/hooks-overview.html) - is a fast-paced overview of the built-in Hooks

[Building Your Own Hooks](https://reactjs.org/docs/hooks-custom.html) - demonstrates code reuse with custom Hooks

[Making Sense of React Hooks](https://medium.com/@dan_abramov/making-sense-of-react-hooks-fdbde8803889) - explores the new possibilities unlocked by Hooks

[useHooks.com](https://usehooks.com/) - showcases community-maintained Hooks recipes and demos



