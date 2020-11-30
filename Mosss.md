# Managing Observable Subscriptions with Synchronous Streams

## Preamble 

This came from a question I asked on Stackoverflow. I just finished writing up my query when the answer came to me. It's an interesting problem because it centers around how synchronous and asynchronous observables work.

To start with, consider these snippits

----
```JavaScript
console.log("Before Timeout");
setTimeout(
  () => console.log("Asynchronous Timeout"), 
  0
);
console.log("After Timeout");
// Output: "before timeout" "after timeout" "Asynchronous Timeout"
```
----
```JavaScript
console.log("Before Promise");
new Promise(resolve => 
  resolve("Asynchronous Promise")
).then(console.log);
console.log("After Promise");
// output: "before promise" "after promise" "Asynchronous Promise"
```
---
```JavaScript
console.log("Before Observable");
new Observable(observer => 
  observer.next("Synchronous Observable")
).subscribe(console.log);
console.log("After Observable");
// output: "before observable" "Synchronous Observable" "after observable"
```
----

Even though the call to `setTimeout` is set to run after 0 milliseconds, it still goes after the current synchronous block is completely done. That's because the code in `setTimeout` is run asynchronously. It goes into JavaScript's event loop and the soonest that it will be executed is after the currently running block of code completes. Promises work on the same premise, even though the promise above resolves immediately it is run asynchronously.

**Observables run synchronously**. They appear asynchronous when combined with any of JavaScript's other asynchronous constructs (ex: `setTimeout`) or with certain RxJS operators (ex: `delay`). In general, however, there is some amount of synchronous execution that happens the moment you `subscribe()` to an Observable.

### The Problem

If an observable is running synchronously, then the callback that is given to `subscribe` is executed **before** `subscribe` returns. The result is that the following code gives an error. `Cannot access 'sub' before initialization`

```JavaScript
const sub = from([1,2,3,4,5]).subscribe(x => {
  if(x > 3) sub.unsubscribe();
  console.log(x);
});
```

### A Nieve Solution

If we force the values of our stream into the event loop, we no longer have this problem. `Subscribe` will always return before the lambda is called.

```JavaScript
const sub = from([1,2,3,4,5]).pipe(
  delay(0)
).subscribe(x => {
  if(x > 3) sub.unsubscribe();
  console.log(x);
});
```

This, however, strikes me as a bad idea. If for no other reason than performance. Though it also makes the run-order less deterministic (Which Browser?, NodeJS?).

### The Idiomatic RxJS Solution

Don't unsubscribe yourself, let an operator do that for you

```JavaScript
const unsub = new Subject();
from([1,2,3,4,5]).pipe(
  takeUntil(unsub)
).subscribe(x => {
  if(x > 3) {
    unsub.next();
    unsub.complete();
  }
  console.log(x);
});
```

The problem here is that we need to create the entire apparatus that is a subject in order to accomplish a very specific goal. It's like buying a truck in order to get the wheel. It doesn't scale well. Finally, just like calling `unsubscribe()` yourself, It's also still mixing imperative and functional javascript.

For this example problem, `take(4)` or `takeWhile(x => x > 3)` both get the job done without the need to `unsubscribe` explicitly. This is ideal where possible.

### The Same Problem at a Bigger Scale

Consider an operator that takes a list of observables and emits values only from the observables earlier in the list than any previous emissions.

Here is this operator done with the event loop.

```JavaScript
function prefer<T, R>(...observables: Observable<R>[]): Observable<R>{
  return new Observable(observer => {

    const subscrptions = new Array<Subscription>();
    const unsub = (index) => {
      for(let i = index; i < subscrptions.length; i++){
        subscrptions[i].unsubscribe();
      }
    }

    observables.map(stream => stream.pipe(
      delay(0) // <-- Delayed Values are placed in the event loop  
    )).forEach((stream, index) => 
      subscrptions.push(stream.subscribe(payload => {
        observer.next(payload);
        unsub(index + 1);
        subscrptions.length = index + 1;
      }))
    );

    return { unsubscribe: () => unsub(0) }
  })
}
```

and then without the event loop and without `unsubscribe()`.

```JavaScript
function prefer<T, R>(...observables: Observable<R>[]): Observable<R>{
  return defer(() => {

    const wUnsub = observables.map((stream, index) => ({
      stream: stream.pipe(
        map(payload => ({index, payload}))
      ), 
      unsub: new Subject()
    }));

    const unsub = (index) => {
      for(let i = index; i < wUnsub.length; i++){
        wUnsub[i].unsub.next();
        wUnsub[i].unsub.complete();
      }
    }

    return merge(...wUnsub.map(build => build.stream.pipe(
      takeUntil(build.unsub)
    ))).pipe(
      tap(({index}) => {
        unsub(index + 1);
        wUnsub.length = index + 1;
      }),
      map(({payload}) => payload),
      finalize(() => unsub(0))
    );
  });
}
```

Also here's the operator in use

```JavaScript
prefer(
  interval(10000).pipe(
    take(5),
    map(_ => "Every 10,000")
  ),
  interval(5000).pipe(map(_ => "Every 5,000")),
  interval(1000).pipe(map(_ => "Every 1,000")),
  interval(250).pipe(map(_ => "Every 250"))
).subscribe(console.log);
```

Imagine using this operator at scale. It's relatively easy to understand that the memory footprint of the first approach is much smaller than the second approach (O(n) vs O(n*n) memory usage).

----

### Finally; The Question

Since (in javascript) synchronous code runs to completion before any other code runs, it doesn't seem to make sense to be able to access an observable's `subscription` before the synchronous section of that subscription has returned. Yet, as a means to abort a stream early, it seems that being able to access a stream's `subscription` early might have benefits (at the very least in memory).

Is there a (relatively) elegant way to instrument an Observable to work around these problems?

## A Solution

It turns out RxJS comes with a pretty good built-in solution. It's only pretty good because it uses publish/connect which seems to be implemented with subjects internally.

This is not really the intended use of publish/connect, as I'm not multicasting. The key is that `ConnectableObservables` do not start with `subscribe`, but rather with `connect`.

You can use this to get at the desired behavior without relying on the event loop at all.

### Solution Using Publish

Mini-example:

```JavaScript
const stream = publish()(from([1,2,3,4,5]));

const sub = stream.subscribe(x => {
  if(x > 3) sub.unsubscribe();
  console.log(x);
});

stream.connect();
```

Scaled to the custom operator:

```JavaScript
function prefer<T, R>(...observables: Observable<R>[]): Observable<R>{
  return new Observable(observer => {

    const subscrptions = new Array<Subscription>();
    const unsub = (index = 0) => {
      for(let i = index; i < subscrptions.length; i++){
        subscrptions[i].unsubscribe();
      }
    }

    observables
      .map(stream => publish()(stream))
      .forEach((stream, index) => {
        subscrptions.push(stream.subscribe((payload: R) => {
          observer.next(payload);
          unsub(index + 1);
          subscrptions.length = index + 1;
        }));
        stream.connect();
      });

    return { unsubscribe: () => unsub() }
  })
}
```

### The Difference:

Using the event loop as follows:

```JavaScript
const stream = from([1,2,3,4,5]).pipe(
  delay(0)
);

console.log("before subscribe");
const sub = stream.subscribe(x => {
  if(x > 3) sub.unsubscribe();
  console.log(x);
});
console.log("after subscribe");
```

The output: `"before subscribe" "after subscribe" 1 2 3 4`

As apposed to using connect as follows:

```JavaScript
const stream = publish()(from([1,2,3,4,5]));

const sub = stream.subscribe(x => {
  if(x > 3) sub.unsubscribe();
  console.log(x);
});

console.log("before connect");
stream.connect();
console.log("after connect");
```

The output: `"before connect" 1 2 3 4 "after connect"`

Because connect keeps synchronous observables, unsubscribe can still happen synchronously and the entire observables stream is processed before the line after connect is run. That's a pretty big win.
