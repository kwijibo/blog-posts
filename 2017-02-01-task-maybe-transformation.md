---
title: "Working with Tasks containing Maybes: another empty Future"
date: "2017-02-01"
readNext: "/fallbacks-alt/"
path: "/task-maybes/"
---

In a previous article, we looked at how to work with a [Task inside a Maybe](/maybe-empty-task/), which is useful when you only want to do some asynchronous operation if you have a value.

A variation on this is a `Task` containing a `Maybe`, which is useful for describing, for example: 

* a database query that may return an empty result (eg `findById('84hjk6h')`)
* an HTTP request to a REST endpoint that  might return a 404
* reading a file that might contain text, or might be empty

The Maybe inside the Task gives us a way to only do something if the result is wrapped in `Just`. If the result is `Nothing`, we do nothing. 

If we have:
```
const result = findById(realID)
//=> Task.of(Just({name: "Robin", img: "343592.jpg"}))
```
we want to transform the object to HTML, and insert it into the page.

If we have:

```
const result = findById(nonexistentID)
//=>Task.of(Nothing())
``` 
We don't want to do _anything_.

The tricky thing with this structure is that if we want to transform the inner value, we have to map twice to get at it:
```
result.map(maybe => maybe.map(transformValue))
```
which would be terrible.

And if we unwrap the Maybe with `.getOrElse(value)` we have to provide a default value, and then we map directly over either the result if it exists, or the default if it doesn't. But we don't want to do anything if we have a Nothing. What we really want is to turn our outer Task into an empty Task if it contains a Nothing, and a Task containing a result if it contains a Just of a result. 

We can achieve this by transforming our Maybe into a Task, and using `chain` with that transformation to ensure that we flatten `Task(Task(result))` into `Task(result)`:

```
const maybeToTask = m => m.map(Task.of).getOrElse(Task.empty())
maybeToTask(Just(42)) //=> Task(42)
maybeToTask(Nothing()) //=> Task.empty()
```
As we [said previously](/maybe-empty-task/), `Task.empty()` returns a Task that will ignore the error and success handlers you pass to `.fork(err, success)` and will therefore do nothing at all.

```
result // Task(Just(result)) or Task(Nothing())
.chain(maybeToTask) //Task(result) or Task.empty()
.map(userToHTML)
.fork(
    handleError,
    renderUser //won't be called if result is Task(Nothing())
)
```

(Note, you won't _always_ want to ignore `Nothing`s - sometimes you'll want to use a default value, or do something completely different.)





