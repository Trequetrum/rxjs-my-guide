# The intervalArray Operator

Takes arrays emitted by the source and spaces out their values by the given interval time in milliseconds

``` JavaScript
function intervalArray<T>(intervalT = 1000): OperatorFunction<T[], T> {
  return s => s.pipe(
    concatMap((v:T[]) => concat(
      ...v.map((value:T) => EMPTY.pipe(
        delay(intervalT),
        startWith(value)
      ))
    ))
  );
}
```
