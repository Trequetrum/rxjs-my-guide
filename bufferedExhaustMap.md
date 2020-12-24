# The `bufferedExhaustMap` operator

This operator buffers values based on a minimum buffer length (time in milliseconds) and minimum buffer size. It always exhausts the current projected observable regardless of whether minimum time and buffer sizes have been met.

This works very much like exhaustMap, except it buffers values instead of discarding values while waiting for the current inner (projected) observable to complete.

## When To Use

This is a way of dealing with backpressure. If the complexity of some process is less than linear (like the binary search algorithm), then a larger input (buffer in this case) is more efficient. The most efficient thing you can do is run the process once on the entire input data set. Unfortunately, if the data is arriving over the course of an hour, day, or forever, then you can't wait for the entire input before starting to process. Especially in long running processes (a server, perhaps), the intermediate values are important.

Even if you're just interacting with a server from a javascript client in a browser (a common, Angular/Vue/React use case), if that server provides a batch request API, then you need a way to know when to batch a request to the server.

A common way to deal with this is to buffer values for a set amount of time or to buffer a certain number of values. The RxJS operators `bufferCount` and `bufferTime` can do that for you. The problem is that if the buffer is too small, you may end up with uncontrollable back pressure but if the buffer is too big, you lose responsiveness.

`bufferedExhaustMap` lets you set a small reactive buffer that dynamically grows based on how long the mapped inner observable takes to complete. That inner observable can encompass a computationally heavy process, read/write to disk, and/or Http call. `bufferedExhaustMap` will buffer until the call is complete.

### First Version

This version creates a custom observable that buffers all values from the source and sets a behaviour subject (`idle`) true/false, so that the next call to `project: () => Observable` will never start until the previous one completes.

There is a lot that could have been done here to be more concise and readable, but it turns out there's a better approach to doing this. I've left this here for posterity and because I liked the way it buffered values. 

```JavaScript
function bufferedExhaustMap<T,R>(
  project: (v:T[]) => ObservableInput<R>, 
  minBufferLength = 0, 
  minBufferCount = 1
): OperatorFunction<T,R> {
  return source => new Observable(observer => {

    const idle = new BehaviorSubject<boolean>(true);
    const setIdle = (b) => () => idle.next(b);

    const buffer = (() => {
      const bufS = new Subject<(v:T[]) => T[]>();
      return {
        output: bufS.pipe(
          scan((acc, curr) => curr(acc), [])
        ),
        nextVal: (v:T) => bufS.next(curr => [...curr, v]),
        clear: () => bufS.next(_ => [])
      }
    })();

    const subProject = combineLatest(
      idle,
      idle.pipe(
        filter(v => !v),
        startWith(true),
        switchMap(_ => timer(minBufferLength).pipe(
          mapTo(true),
          startWith(false)
        ))
      ),
      buffer.output
    ).pipe(
      filter(([idle, bufferedByTime, buffer]) => 
        idle && bufferedByTime && buffer.length >= minBufferCount
      ),
      tap(setIdle(false)),
      tap(buffer.clear),
      map(x => x[2]),
      map(project),
      mergeMap(projected => from(projected).pipe(
        finalize(setIdle(true))
      ))
    ).subscribe(observer);

    const subSource = source.subscribe({
      next: buffer.nextVal,
      complete: observer.complete.bind(observer),
      error: e => {
        subProject.unsubscribe();
        observer.error(e);
      }
    });

    return {
      unsubscribe: () => {
        subProject.unsubscribe();
        subSource.unsubscribe();
      }
    }
  });
}
```

---

Aside: In the function parameters, ObservableInput works with an Array, an array-like object, a Promise, an iterable object, or an Observable-like object.

### Second Version

This version is much nicer. For one, it's entirely functional composition using RxJS operators. Much of the work in the previous version was to make sure the operator cleaned up after itself. That's not a concern since we're not constructing an observable from scratch.

Another huge difference between the two versions is that in this newer version we subscribe to the source over and over and over again. This is done so that `bufferWhen` can do its job (We built a custom buffer in the previous version). To do this without causing unexpected behaviour, we have to multicast the source. This is done with `share`.

Multicasting comes with a cost, but our custom buffer was a subject which also multicasts, so there's not really a performance hit when compared with the previous version.

Here's the operator:

```JavaScript
/***
 * Buffers, then projects buffered source values to an Observable which is merged in 
 * the output Observable only if the previous projected Observable has completed.
 ***/
function bufferedExhaustMap<T,R>(
  project: (v:T[]) => ObservableInput<R>, 
  minBufferLength = 0, 
  minBufferCount = 1,
  concurrent = 1
): OperatorFunction<T,R> {
  return source => defer(() => {

    /***
     * Outputs/emits the numberical difference between how often
     * start and end have been called. Tracks concurrently running
     * instances o the projected observable.
     ***/
    const projectCount = (() => {
      const projecting = new Subject<boolean>();
      return {
        output: projecting.pipe(
          scan((acc, curr) => curr? ++acc : --acc, 0),
          startWith(0)
        ),
        start: () =>  projecting.next(true),
        end: () =>  projecting.next(false)
      }
    })();

    // Multicast the source so we can use it to manage our buffer
    const shared = source.pipe(share());

    // observable that emits a 1 when the buffer 
    // should be cleared, then completes.
    const nextBufferTime = () => merge(
      // Take minBufferCount from source then complete
      shared.pipe(take(minBufferCount)),
      // Wait minBufferLength milliseconds and then complete
      timer(minBufferLength),
      // Wait current projected observables drops below the given
      // concurrent count.
      projectCount.output.pipe(first(count => count < concurrent))
    ).pipe(
      // Ignore all values, only care about when they complete
      filter(_ => false),
      // Emit 1 after merged observables complete
      endWith(1),
      // projectCount.start before clearing buffer
      tap(projectCount.start)
    );

    return shared.pipe(
      bufferWhen(nextBufferTime),
      // map (v:T[]) => ObservableInput<R>
      map(project),
      // Turn ObservableInput into Observable, then 
      // projectCount.end once it's complete
      mergeMap(projected => from(projected).pipe(
        finalize(projectCount.end)
      ))
    );
  });
}
```

Perhaps the most complex bit is the `nextBufferTime` factory function. It creates an observable that emits a 1 when the buffer should be cleared, then completes.

It does this by merging 3 observables and only completing once all three observables are complete. In this way, an inner observable completing counts as a condition being met.

 - *First condition*: The source has emitted `minBufferCount` values. 
 - *Second condition*: A timer of `minBufferLength` has elapsed.
 - *Third condition*: No projected observable is currently active (`idle == true`).

```JavaScript
const nextBufferTime = () => merge(
  // Take minBufferCount from source then complete
  shared.pipe(take(minBufferCount)),
  // Wait minBufferLength milliseconds and then complete
  timer(minBufferLength),
  // Wait until idle emits true, then complete
  idle.pipe(first(v => v))
).pipe(
  // Ignore all values, we only care about when they complete
  filter(_ => false),
  // Emit 1 after merged observables complete 
  endWith(1)
);
```


