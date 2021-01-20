# The intervalArray Operator

Takes arrays emitted by the source and spaces out their values by the given interval time in milliseconds

``` JavaScript
function intervalArray<T>(intervalT = 1000): OperatorFunction<T[], T> {
  return s => s.pipe(
    concatMap(a =>
      zip(from(a), timer(0, intervalT)).pipe(
        map(([x, _]) => x),
        s => concat(s, timer(intervalT).pipe(
          filter(_ => false)
        ))
      ) as Observable<T>
    )
  );
}
```
