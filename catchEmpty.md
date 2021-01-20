# The catchEmpty Operator

This operator functions very similarly to [catchComplete](catchComplete.md), only it will only catch a completed stream that has never emitted any values.

```JavaScript
function catchEmpty<T, R>(fn: () => Observable<R>): OperatorFunction<T, T|R> {
  return s => defer(() => {
    let count = 0;
    return concat(
      s.pipe(tap(() => count++)),
      defer(() => count === 0? fn() : EMPTY)
    );
  });
}
```
