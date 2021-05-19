# Concurrency Limiter for Disparate Streams

```TypeScript
export class ConcurrencyLimiter {
  private currentStreamId = 0;
  private starter = new Subject<{ id: number; obs: Observable<any> }>();
  private run$: Observable<any>;

  constructor(concurrent = 1) {
    this.run$ = this.starter.pipe(
      delay(0),
      mergeMap(
        ({ id, obs }) => obs.pipe(map(payload => ({ id, payload }))),
        concurrent
      ),
      shareReplay(1)
    );
  }

  constrain<T>(stream: Observable<T>): Observable<T> {
    return defer(() => {
      const currentId = this.currentStreamId++;
      return this.run$.pipe(
        initialize(() => this.starter.next({ id: currentId, obs: stream })),
        first(({ id }) => id === currentId),
        map(({ payload }) => payload)
      );
    }) as Observable<T>;
  }

  limit<T>(): MonoTypeOperatorFunction<T> {
    return this.constrain.bind(this);
  }
}
```

Another approach:

```TypeScript
export class ConcurrencyLimiter2 {
  private currentStreamId = 0;
  private starter = new Subject<{ start: boolean; id: number }>();
  private control$: Observable<number>;
  private runCount = 0;
  private buffer: number[] = [];

  constructor(concurrent = 1) {
    this.control$ = this.starter.pipe(
      tap(({ start }) => {
        if (!start) this.runCount--;
      }),
      filter(({ start, id }) => {
        if (this.runCount < concurrent) return true;
        if (start) this.buffer.unshift(id);
        return false;
      }),
      tap(({ start }) => {
        if (start || this.buffer.length > 0) this.runCount++;
      }),
      map(({ start, id }) => (start ? id : this.buffer.pop())),
      filter(v => v != null),
      shareReplay(1)
    );
  }

  constrain<T>(stream: Observable<T>): Observable<T> {
    return defer(() => {
      const currentId = this.currentStreamId++;
      return this.control$.pipe(
        initialize(() =>
          this.starter.next({
            start: true,
            id: currentId
          })
        ),
        first(id => id === currentId),
        switchMapTo(stream),
        finalize(() =>
          this.starter.next({
            start: false,
            id: currentId
          })
        )
      );
    });
  }

  limit<T>(): MonoTypeOperatorFunction<T> {
    return this.constrain.bind(this);
  }
}
```
