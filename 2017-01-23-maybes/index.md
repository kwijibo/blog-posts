---
title: "Programming with Uncertain Values"
date: "2017-01-23T22:40:32.169Z"
path: "/uncertain-values/"
---

A common pain in programming is when you need to do something with a value that might not exist. You have to wrap all the code that depends on that value inside an `if` statement checking for `null`.  It's a little bit like using node.js-style callbacks; everything that needs the value has to be nested inside the callback: 

```
const getUserName = () => {
    const data = window.localstorage.getItem('user')
    if(data){
        const user = JSON.parse(data)
        if(user.hasOwnProperty('name')){
            return user.name
        }
    }
}
```
Pyramid of doom!

And the uncertainty propagates out to any code that calls `getUserName`, because it might not return a user name.

What we want is an abstraction that lets you do stuff with a value, whether it exists or not, without having to wrap everything in `if`s at each step. 

### A context for certainty

An abstraction that does this, is [Array.prototype.map](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Array/map). You pass a function into `map` and it will do stuff to the contents of the array, whether they exist or not. `[].map(f)` doesn't run `f` and is perfectly safe, no `if`s required. 

So we can do:

```
const localStorageGet = (key) => {
    const data = window.localStorage.getItem(key)
    return data? [data] : []
}
const getUser = () => 
    localStorageGet('user')
    .map(JSON.parse)
```

Now whenever we want to do something with our (potential) user object, we can just map a function over the array containing it. If the array is empty, the functions don't get run.
Any transformation we need to run can be isolated in discrete functions, and they don't need to check for `null`/`undefined`. We only need to remember to deal with the possibility that we never had a user in the first place when we unpack the value from our containing `[]`:

```
const userName = getUser().map(u => u.name)

//now we unpack it, with a default if it doesn't exist.
const displayName = userName.length? userName[0] : 'Guest' 
```

We've got something pretty nice with this `map` abstraction.  We can compose functions over our potential value without needing to null check in each function.

```
f(g(h(undefined)))
```

would blow up on us if  `h` `g` or `f` don't expect their argument to be `undefined`, but

```
[].map(h).map(g).map(f)
```

won't run `h` `g` or `f` because the array doesn't have a value to run them over.

However, it _is_ a bit hacky to overload Array in this way, because, for example:
1. How do you tell if `[x]` is meant to be an array or a "potential value"?
2. If you want to represent a _potential_ `false`, `null`, or `undefined`, you have to be careful how you check whether the value exists
3. If you `map` a function that returns a potential value, over a potential value, you get either `[]`, `[[]]` or `[[x]]`, which can be a little fiddly to map over or unpack.

### Maybe: A datatype for values that might not exist

What we _really_ want is a datatype designed to represent values that might not exist. Many functional languages and libraries have this type, and it's called `Maybe`. You can find good implementations of this in libraries like [folktale](http://folktalejs.org/), [ramda-fantasy](https://github.com/ramda/ramda-fantasy/) or [sanctuary](http://sanctuary.org/), but we'll define a simple version here to see how it works.

```
const Just  = x => ({
    map: f => Just(f(x)),
    chain: f => f(x),
    getOrElse: fallback => x
})
const Nothing = () => ({
    map: f => Nothing(),
    chain: f => Nothing(),
    getOrElse: fallback => fallback
})
```

`Just` and `Nothing` are two types that share the same `Maybe` interface. A `Maybe` is something that could be a `Just` or a `Nothing`; we don't need to care which we have, because they both have the same interface, and it will all just work.

`Just(x)` is what we use instead of `[x]`. `Just(1).map(x =>x+1)` works just the same as `[1].map(x=>x+1)`. `Just(a).map(f)` is equal to `Just(f(a))`

`Nothing().map(x=>x+1)` works the same as `[].map(x=>x+1)` - ie, you just get another `Nothing` back. 

`getOrElse` will give us either a value (from a `Just`) or the provided default (if we've got a `Nothing`):

```
Just(3).getOrElse(5) //=> 3
Nothing().getOrElse(5) //=> 5
```

`chain` lets us compose in functions that themselves return *Maybe*s rather than ordinary values - we'll see an example of this shortly.

### How we use Maybe in practice

A common use for Maybes is when you want to access an object property that might not exist:

```
//:: String -> Object -> Maybe value
const safeProp = prop => 
    obj => 
    obj.hasOwnProperty(prop)? Just(obj[prop]) : Nothing()

```

An example:
```
const data = { contact: { email: "abcd@example.org" } }
safeProp('contact')(data) //=> Just({ email "abcd@example.org"})
safeProp('contact')({}) //=> Nothing()
```
But what if we want to safely access `data.contact.email`? 

```
safeProp('contact')(data).map(safeProp('email'))
```
would give us `Just(Just("abcd@example.org"))` which would be fiddly to map over or unpack. 

This is why we've got `chain`:

```
safeProp('contact')(data)
    .chain(safeProp('email'))
```
gives us simply `Just("abcd@example.org")`

If you're familiar with Promises, you might be seeing a parallel here: *Promise* lets you do something with a value _when_ it exists; *Maybe* lets you do something with a value _if_ it exists.

Promises let you flatten out nested callbacks (also known as "callback hell"), Maybes let you flatten out nested `if`s ("uncertainty hell"?):

```
const getUserName = (fallback='Guest') => {
    const data = window.localstorage.getItem('user')
    if(data){
        const user = JSON.parse(data)
        if(user.hasOwnProperty('name')){
            return user.name
        } 
    }
    return fallback
}
```
Can become

```
const getMaybeUserName = () =>
    localStorageGet('user')
    .map(JSON.parse)
    .chain(safeProp('name'))
```

### Why is this better?

There are some downsides to the Maybe version: I've had to invent some custom types and functions to support it, and readers of the code may have trouble figuring out what it does compared to low level vanilla javascript. In some situations, an `if` may be the best choice.

However, the `if` version either hides the uncertainty from the caller by returning a default, or propagates it by returning a null. 
By returning a *Maybe*, `getMaybeUserName` is honest about the uncertainty and gives the caller control. We can carry on composing operations into our Maybe, or extract a value when we choose:

```
// render a greeting to the user, 
// using 'Guest' as the default username

const maybeUserName = getMaybeUserName()
renderGreeting(maybeUserName.getOrElse('Guest'))

// If we have a username
// use it to lookup a user
// then if the user has a 'favs' list , render it

const maybeRenderFavs = maybeUserName
.map(lookupUser) //=> Maybe(Promise({favs: [Fav]}))
.getOrElse(Promise.resolve({})) //=> Promise({})
.then(u => 
    safeProp('favs')(u)
    .map(renderFavourites)
)
.catch(handleErrors)
``` 

NB: While I've used `Promise`s in the example above, there is another abstraction for asychronous values called [Task](http://docs.folktalejs.org/en/latest/api/data/task/Task.html) (also known as [Future](https://github.com/ramda/ramda-fantasy/blob/master/docs/Future.md) in some libraries) which can have some [advantages over Promises](https://glebbahmutov.com/blog/difference-between-promise-and-task/).

The `map` and `chain` interfaces give you *composability* and  *caller control*, not just with uncertain values or asynchronous values, but for data types with other semantics as well - see [Ramda Fantasy](https://github.com/ramda/ramda-fantasy/) for a sample. 

There are also other generic interfaces that can be implemented across various different datatypes - [Fantasy Land](https://github.com/fantasyland/fantasy-land) is a specification that defines a selection of the most useful of these.

For further explanation, I highly recommend [@drboolean](http://twitter.com/drboolean)'s [Professor Frisby Videos](https://egghead.io/courses/professor-frisby-introduces-composable-functional-javascript) and [online book, Mostly Adequate Guide to Functional Programming](https://drboolean.gitbooks.io/mostly-adequate-guide/) 
