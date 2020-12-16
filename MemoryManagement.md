### Memory Management

You will often hear: “If you subscribe to a stream, you must eventually unsubscribe or you risk a memory leak”

Which leads into some pretty nonsensical “always unsubscribe” mantras.

**Here’s the actual rule:** Streams that *must* eventually complete or error **do not** need to be unsubscribed. Streams that do not (or *may* not) complete or error eventually **do** need to be unsubscribed.

Every native RxJS operator has subscription management built-in.

This is part of RxJS’s design. If you’re not multicasting, you can compose pure streams of any complexity with RxJS’s operators. When that stream errors, completes, or is unsubscribed, you can be sure every stream created by every intermediate operation is managed. They’re all done and no silent `interval`/`timer` will be left firing into the void and accomplishing nothing.

---

### A Simple Example

Here I create `interval-1` and `interval-2`. Interval-1 emits every second and interval-2 emits every two seconds. They count from 1 upwards for forever.

If you run this, this will keep emitting values until you end the program:

```JavaScript
of(1,2).pipe(
  mergeMap(x => interval(x*1000).pipe(
    map(val => `interval-${x}: ${val + 1}`)
  ))
).subscribe(console.log);
```

The first 10 seconds of output:

```
interval-1: 1
interval-1: 2
interval-2: 1
interval-1: 3
interval-1: 4
interval-2: 2
interval-1: 5
interval-1: 6
interval-2: 3
interval-1: 7
interval-1: 8
interval-2: 4
interval-1: 9
interval-1: 10
interval-2: 5
```

If, eventually, I want this stream to stop, I must unsubscribe. Here is how I might unsubscribe after 10 seconds.

```JavaScript
const subscription = of(1,2).pipe(
  mergeMap(x => interval(x*1000).pipe(
    map(val => `interval-${x}: ${val + 1}`)
  ))
).subscribe(console.log);

setTimeout(
  () => subscription.unsubscribe(), // unsubscribe
  10 * 1000 // after 10 seconds
);
```

But there’s another way! If we know that we want to unsubscribe after 10 seconds, then we can instead create a stream that completes after 10 seconds. Remember that we don’t have to unsubscribe from streams that complete (and RxJS operators take care of the rest).

If we do the following, we don’t have to worry about unsubscribing: 

```JavaScript
of(1,2).pipe(
  mergeMap(x => interval(x*1000).pipe(
    map(val => `interval-${x}: ${val + 1}`)
  )),
  takeUntil(timer(10 * 1000))
).subscribe(console.log);
```

### Explaining How `take(10)` Works

You've just seen `takeUntil` at work. `take(n)` is similar. Once it's seen `n` emissions, it stop the stream. I'll explain how that works and why we no longer need to unsubscribe from anything.

Here's what the example looks like:

```JavaScript
of(1,2).pipe(
  mergeMap(x => interval(x*1000).pipe(
    map(val => `interval-${x}: ${val + 1}`)
  )),
  take(10)
).subscribe(console.log);
```

Here's the magic, `take(10)` will wait for 10 emissions from the `mergeMap`, then it will unsubscribe from the `mergeMap`. After that, it emits `complete` (yay, we have a stream that completes - subscription worries abated). 

The story, however, doesn't stop just yet. This stream is composed of more than just one operator. Before `take(10)` is a `mergeMap`.

When `mergeMap` is unsubscribed, it unsubscribes from all it’s inner observables (`interval-1` and `interval-2`). Normally, it also unsubscribes from its source observable (in this case, that’s `of(1,2)`), but this time the source observable completed already.

Interval stops emitting once unsubscribed. It's a creation operator and doesn't/can't have any source oservables to unsubscribe to. So they're pretty simple to manage.

In this way, each operator knows what it needs to do in order to keep your code clean and free of memory leaks. This example is pretty straight forward. `MergeMap` is the most complex operator in this pipeline as it:

- Manages 3 observables: A source observable and two emissions from the source creating  2 new inner observables (`interval-1` and `interval-2`)
- If unsubscribed, must unsubscribe from all managed observables that are still active.
- If any of the 3 observables error, the other two must be unsubscribed (if active) and the error emitted
- If the source completes, it must be ready to complete once the inner observables both complete.

If you have 5 mergeMaps back-to-back

```JavaScript
stream.pipe(
  mergeMap(/** something **/),
  mergeMap(/** something-else **/),
  mergeMap(/** something-fun **/),
  mergeMap(/** something-wierd **/),
  mergeMap(/** something-redundant **/)
)
```

They can each blindly worry just about their inner and source observables and manage unsubscribe calls properly and the whole chain of events is well managed. This is all part of the magic of RxJS. It’s not just mergeMap that does this, every operator does. So as you compose operators and know that the composed operators continue to keep these properties.

### Example Memory Leak

The following code will put the same output to the console as the `take(10)` example above, but since subscribe doesn’t manage source observable subscription, it will create a memory leak.

```JavaScriipt
of(1,2).pipe(
  mergeMap(x => interval(x*1000).pipe(
    map(val => `interval-${x}: ${val + 1}`)
  )),
  map((inter, index) => ({inter, index}))
).subscribe(({inter, index}) => {
  if(index < 10){
    console.log(inter);
  }
});
```

To a user this might look the same as `take(10)`, as the first 10 values are printed to the console and then nothing else seems to happen.

If you want to see the memory leak in action, we can add a tap into our stream and write something to the console. We'll write the word "poke".

```JavaScriipt
of(1,2).pipe(
  mergeMap(x => interval(x*1000).pipe(
    map(val => `interval-${x}: ${val + 1}`)
  )),
  map((inter, index) => ({inter, index})),
  tap(_ => console.log("poke"))
).subscribe(({inter, index}) => {
  if(index < 10){
    console.log(inter);
  }
});
```

Now you'll notice that "poke" keeps being written to the console for forever.

Some of the output:

```JavaScript
poke
interval-1: 1
poke
interval-1: 2
poke
interval-2: 1
poke
interval-1: 3
poke
interval-1: 4
poke
interval-2: 2
poke
interval-1: 5
poke
interval-1: 6
poke
interval-2: 3
poke
interval-1: 7
poke
poke
poke
poke
poke
// ... 10 minutes later
poke
poke
poke
poke
```

# Never Unsubscribe

Also, never say never

In general, I would argue it’s a good idea to always create streams that complete eventually. Sometimes the way this can be accomplished is not too clear at first, but there are often certain states of your program whose logic you can hook into to signal the end of a long-running stream.

If you’re processing user clicks on the DOM, you can `takeUntil` the user navigates to a new page.

The more event-drive your program is, the more intuitive this becomes. 

On the other hand, here’s a pattern I see all the time:

```JavaScript
const endSignal$ = new Subject();

interval(1000).pipe(
  takeUntil(endSignal$)
).subscribe(console.log);

setTimeout(
  () => {
    endSignal$.next();
    endSignal$.complete();
  },
  10 * 1000
);
```

I’m not sure how I feel about this. Remember how I unsubscribed in the simple example earlier? It looked very similar: 

```JavaScript
const subscription = interval(1000).subscribe(console.log);

setTimeout(
  () => subscription.unsubscribe(),
  10 * 1000
);
```

Of course, the code inside the `setTimeout` callback can exist anywhere for either example. It’s done like this for brevity’s sake.

All the `endSingnal$` subject adds is the memory footprint and management of a subject. It’s like buying a car when all you needed was the wheel. However, if `endSingnal$`is being used to end a number of streams, you’re being saved the bother of managing a list of subscriptions and then iterating through them to unsubscribe.

It’s also a pattern that lets you complete inner observables within a mergeMap (for example) without affecting the mergeMap itself (which can be tricky without this pattern). Where multiple streams are concerned, it’s also much clearer and cleaner to see what is happening and how a stream completes (or is unsubscribed). 

In the end, however, it seems this can be a matter of taste.

The more functional and event-driven your program is, the less you’ll run into this choice. 


