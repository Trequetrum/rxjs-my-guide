# The catchComplete operator

This operator functions very similarly to `catchError`, only it functions over `complete` instead of `error` messages. 
This lets you chain a new observable into the completion of a previous observable.

```JavaScript
function catchComplete<T, R>(fn: () => Observable<R>): OperatorFunction<T, T|R> {
  return s => new Observable(observer => {
    let iSub: Subscription;
    const oSub = s.subscribe({
      next: x => observer.next(x),
      error: err => observer.error(err),
      complete: () => {
        iSub = fn().subscribe(observer);
      }
    });
    return { unsubscribe: () => {
      oSub?.unsubscribe();
      iSub?.unsubscribe();
    }}
  });
}
```

## Updated Implementation:


```JavaScript
function catchComplete<T, R>(fn: () => Observable<R>): OperatorFunction<T, T|R> {
  return s => concat(s, defer(fn));
}
```

### catchComplete in use:

```JavaScript
from([1,2,3,4,5]).pipe(
  catchComplete(() => from([5,4,3,2,1]))
).subscribe(console.log);
// Output: 1 2 3 4 5 5 4 3 2 1
```
