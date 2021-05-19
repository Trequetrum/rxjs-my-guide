# The intervalArray Operator

```TypeScript
/****
 * Turn an array into an observable that emits each item 
 * (and complete) with the given interval time in milliseconds.
 ****/
function intervalArry<T>(arry: T[], intervalTime = 1000): Observable<T> {
  return concat(...arry.map((value: T) =>
    EMPTY.pipe(
      delay(intervalTime),
      startWith(value)
    ))
  );
}

/****
 * Pipeable Operator:
 * Takes arrays emitted by the source and spaces out their
 * values by the given interval time in milliseconds
 ****/
function intervalArray<T>(intervalTime = 1000): OperatorFunction<T[], T> {
  return pipe(
    concatMap((v: T[]) => intervalArry(v, intervalTime))
  );
}
```
