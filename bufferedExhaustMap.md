# The `bufferedExhaustMap` operator

This operator buffers values based on a minimum buffer length (time in milliseconds) and minimum buffer size. It always exhausts the current projected observable regardless of whether minimum time and buffer sizes have been met.

This works very much like exhaustMap, except it buffers values instead of discarding values while waiting for the current inner (projected) observable to complete.

### First Version

This version creates a custom observable that buffers all values from the source and sets a behaviour subject (`idle`) true/false, so that the next call to `project: () => Observable` will never start until the previous one completes.

There is a lot that could have been done here to be more concise and readable, but it turns out there's a better appraoch to doing this. I've left this here for posterity and because I liked the way it buffered values. 

```JavaScript
function bufferedExhaustMap<T,R>(
  project: (v:T[]) => ObservableInput<R>, 
  minBufferLength = 0, 
  minBufferCount = 1
): OperatorFunction<T,R> {
  return s => new Observable(observer => {

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

    const subSource = s.subscribe({
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

This version is much nicer. For one, it's entirely functional composition using RxJS operators. Much of the work in the previous version was to make sure the operator cleaned up after itself. That's not a consern since we're not custructing an observable from scratch.

Another huge difference between the two versions is that in this newer version we subscribe to the source over and over and over again. This is done so that `bufferWhen` can do its job (We built a custom buffer in the previous version). To do this without causing unexpected behaviour, we have to multicast the source. This is done with `share`.

Multicasting comes with a cost, but our custom buffer was a subject which also multicasts, so there's not really a performance hit when compared with the previous version.

Here's the operator:

```JavaScript
function bufferedExhaustMap<T,R>(
  project: (v:T[]) => ObservableInput<R>, 
  minBufferLength = 0, 
  minBufferCount = 1
): OperatorFunction<T,R> {
  return s => defer(() => {
    const idle = new BehaviorSubject(true);
    const setIdle = (b) => () => idle.next(b);
    const shared = s.pipe(share());

    const nextBufferTime = () => merge(
      shared.pipe(take(minBufferCount)),
      timer(minBufferLength),
      idle.pipe(first(v => v))
    ).pipe(
      filter(_ => false),
      endWith(1),
      tap(setIdle(false))
    );

    return shared.pipe(
      bufferWhen(nextBufferTime),
      map(project),
      mergeMap(projected => from(projected).pipe(
        finalize(setIdle(true))
      ))
    );
  });
}
```

Perhaps the most complex bit is the `nextBufferTime` factory function. It creates an observable that emits a 1 when the buffer should be cleared, then completes.

It does this by merging 3 observables and only completing once all three observables are complete. In this way, an inner observable completing counts as a condition being met.

- First condition: The source has emitted minBufferCount values. 
- Second condition: A timer of minBufferLength has elapsed.
- Third condition: No projected observable is currently active (idle == true).

```JavaScript
const nextBufferTime = () => merge(
  // Take minBufferCount from source then complete
  shared.pipe(take(minBufferCount)),
  // Wait minBufferLength milliseconds and then complete
  timer(minBufferLength),
  // Wait until idle emits true, then copmlete
  idle.pipe(first(v => v))
).pipe(
  // Ignore all values, we only care about when they complete
  filter(_ => false),
  // Emit 1 after merged observables complete 
  endWith(1)
);
```


