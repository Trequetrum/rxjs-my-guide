# Never Nest Subscriptions

One of the first things you’re likely to read as you’re learning about RxJS is that nesting subscriptions is a code smell. You shouldn’t do it.

Here we'll look at some of the reasons and ways in which people nest subscriptions and ways to do the same without nesting.

## Unnesting subscriptions with mergeMap

---

An Aside:

> In cases where your subscribed observable emits once and completes (like an http request), switchMap and mergeMap produce the same output. switchMap is often recommended over mergeMap in these cases. The reasons range from debugging memory leaks, to marginal performance, to what other developers expect.
>
> For simplicity's sake, I've ignored that here and used mergeMap in all cases.

---

When you first start using RxJS it's pretty intuitive (and tempting) to nest subscriptions.

Lets say you have `function doSomething(): Observable<value>` and you know you can get the value as follows:

```JavaScript
doSomething().subscribe(something => {
  console.log("Here is the name of something: " + something.name);
  console.log("and here is its size: " + something.size);
});
```

You now have a clear place where you have the value you need. It’s between those curly braces `{ … }` inside subscribe’s callback function. 

So what happens when you want to doSomething(), and then use that value as a parameter to `function doSomethingElse(params): Observable<value>`

Here’s the temping way to do that:

```JavaScript
doSomething().subscribe(something => {
  console.log("Here is the length of something: " + something.length)
  doSomethingElse(
    something.name,
    something.size
  ).subscribe(somethingElse => {
    console.log("I got something else now: " + somethingElse.value);
  });
});
```

This is bad and you should feel ashamed. No, it’s not that bad but it is a code smell. Too much of this sort of thing and your code-base starts to become a maze of callbacks. They’re hard to debug and the cognitive load of maintaining and extending your program becomes considerable after a few short months of development. Never mind being part of a bigger team and managing code reviews.

Here’s how you write it without a nested subscription:

```JavaScript
doSomething().pipe(
  mergeMap(something => {
    console.log("Here is the length of something: " + something.length);
    return doSomethingElse(
      something.name, 
      something.size
    );
  })
).subscribe(somethingElse => {
  console.log("I got something else now: " + somethingElse.value);
});
```

You can almost always just return the stream you were going to subscribe to inside `subscribe` to `mergeMap` and have it subscribe for you.

---

### Dealing with Functional Closures

 What about this?

```JavaScript
 doSomething().subscribe(something => {
  doSomethingElse(
    something.name,
    something.size
  ).subscribe(somethingElse => {
    console.log(`I used ${something.name} to get ${somethingElse.name}`);
  });
});
```

Here, you need access to both `something` and `somethingElse` inside the callback. With nested subscriptions you can rely on functional closure to hold those values for you. How can you get those into your subscription otherwise?

Well, the idiomatic way to do this is to map each response that you require in the final subscribe into an object, array, map, or other data structure that the subscription callback can consume.

```JavaScript
doSomething().pipe(
  mergeMap(something => 
    doSomethingElse(something.name, something.size).pipe(
      map(somethingElse => ({something, somethingElse}))
    )
  )
).subscribe(({something, somethingElse}) => {
  console.log(`I used ${something.name} to get ${somethingElse.name}`);
});
```

This actually looks more complicated than just nesting subscriptions at first glance. The power, of course is that you can extend this into a very long series of calls without adding any additional complexity. Continuing to nest subscriptions forever doesn't scale.

If you want to see this done with more calls as a series of `switchMap`, I write about how to solve this problem as part of a pattern here: [The Consecutive HTTP Call Pattern](Chttpcp.md)

### Dealing with `for`/`forEach` Loops

TODO

## Never Say Never 

Nesting subscriptions is a code smell, but of course, like most code smells, there’s a time and a place.

The only thing a nested subscription does that `mergeMap` can’t accomplish (and more) is that it doesn’t manage subscriptions at all. You can unsubscribe from `mergeMap` and it will, in turn, unsubscribe from all the merged streams whose values it is emitting. No chance of a memory leak. 

This is explained with more detail here: [Memory Management](MemoryManagement.md).

Here’s an example problem where the best solution I’ve come across (so far) manages its own subscriptions (and nests `subscribe()` calls 1 level deep): [Managing Observable Subscriptions with Synchronous Streams](Mosss.md)

Short of some corner cases, however, `mergeMap` is a superset of nested subscriptions. So you should really stick with RxJS operators like mergeMap. 


