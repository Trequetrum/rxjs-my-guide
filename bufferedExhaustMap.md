# The `bufferedExhaustMap` operator

This operator buffers values based on a minimum buffer length (time in milliseconds) and minimum buffer size. It always exhausts the current projected observable regardless of whether minimum time and buffer sizes have been met.

This works very much like exhaustMap, except it buffers values instead of discarding values while waiting for the current inner (projected) observable to complete.

### First Version

```JavaScript
function bufferedExhaustMap<T,R>(
  project: (v:T[]) => ObservableInput<R>, 
  minBufferLength = 0, 
  minBufferCount = 1
): OperatorFunction<T,R> {
  return s => new Observable(observer => {

    const idle = new BehaviorSubject<boolean>(true); 
    const buffering = new Subject<(v:T[]) => T[]>();
    const buffered = buffering.pipe(
      scan((acc, curr) => curr(acc), [])
    );

    const subBuff = combineLatest(
      idle,
      idle.pipe(
        filter(v => !v),
        startWith(true),
        switchMap(_ => timer(minBufferLength).pipe(
          mapTo(true),
          startWith(false)
        ))
      ),
      buffered
    ).pipe(
      filter(([idle, bufferedByTime, buffer]) => 
        idle && bufferedByTime && buffer.length >= minBufferCount
      ),
      tap(_ => {
        idle.next(false);
        buffering.next(_ => []);
      }),
      map(([_1, _2, buffer]) => buffer),
      mergeMap((buffer:T[]) => from(project(buffer)).pipe(
        finalize(() => idle.next(true))
      ))
    ).subscribe(observer);

    const subSource = s.subscribe({
      next: v => buffering.next(curr => [...curr, v]),
      complete: observer.complete.bind(observer),
      error: e => {
        subBuff.unsubscribe();
        observer.error(e);
      }
    });

    return {
      unsubscribe: () => {
        subSource.unsubscribe();
        subBuff.unsubscribe();
      }
    }
  });
}
```

### Second Version

```JavaScript
function bufferedExhaustMap2<T,R>(
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
      mergeMap((v:T[]) =>
        from(project(v)).pipe(
          finalize(setIdle(true))
        )
      )
    );
  });
}
```

---

Aside: ObservableInput works with an Array, an array-like object, a Promise, an iterable object, or an Observable-like object.
