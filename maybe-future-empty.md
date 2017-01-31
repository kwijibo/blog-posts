---
title: "How to handle an uncertain Future, with Task.empty"
path: "/maybe-empty-task/"
---
  
In my previous post [introducing Maybe](/uncertain-values/), I ended with an example using Promises, mentioning that [Task](http://docs.folktalejs.org/en/latest/api/data/task/Task.html) can be a useful alternative to javascript Promises. This was the example:

```javascript
getMaybeUserName()
.map(lookupUser) //=> Maybe(Promise({favs: [Fav]}))
.getOrElse(Promise.resolve({})) //=> Promise({})
.then(u => 
    safeProp('favs')(u)
    .map(renderFavourites)
)
.catch(handleErrors)
``` 

A _Task_ version of `lookupUser` might be written like this:

```javascript
//lookupUser:: String -> Task {favs}
const lookupUser = username => 
    new Task((reject, resolve) =>
        fetch(`/users/?name=${username}`)
        .then(r=> resolve(r.json()))
        .catch(reject)
    )
```

And a translation of the example to use the Task API could be:

```
getMaybeUserName() //Maybe String
.map(lookupUser) // Maybe Task Object
.getOrElse(Task.of({})) //Task Object
.map(safeProp('favs')) //Task Maybe Object
.fork(
      handleErrors
    , maybeUser => 
        maybeUser.map(renderFavourites)
)
```
There are a couple of things I don't like about this. First, we've propagated the uncertainty from before the `lookupUser` call, to after it; we're unwrapping our Maybe to a `Task.of({})`, and then we have to handle a possible empty response later, even though our imaginary server only ever returns full user objects, or errors.

Second, I don't like ```maybeUser.map(renderFavourites)``` because `renderFavourites` performs an effect (mutating the DOM), rather than returning a value, and functions passed to `.map` should return a value, not perform effects. A stricter language than Javascript would not let us be so hacky.

We can solve these problems by making our Maybe default to an _empty Task_, instead of an empty Object inside a Task. An empty Task is simply `new Task(()=>{})`. You can get one by calling `Task.empty()`, and:

```javascript
Task.empty()
.fork(
 console.error,
 console.log
)
```
won't do anything, because the empty Task doesn't call either of the functions passed to `.fork`. So we can rewrite the example:

```
getMaybeUserName()
.map(lookupUser)
.getOrElse(Task.empty())
.map(d => d.favs)
.fork(
      handleErrors
    , renderFavourites
)
```




