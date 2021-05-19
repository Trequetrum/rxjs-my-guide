# Predicate Throttle Time

```TypeScript
function throttleTimeOn<T>(
  thrTime: number,
  pred: (x:T) => boolean
): MonoTypeOperatorFunction<T> {
  return s => defer(() => {
    let throttleTimeStamp = 0;
    return s.pipe(
      filter(v => !pred(v)?
        true:
        Date.now() - throttleTimeStamp >= thrTime
      ),
      tap(v => { if(pred(v)) {
        throttleTimeStamp = Date.now();
      }})
    );
  });
}
```

# priorityThrottleTime

```TypeScript
function priorityThrottleTime<T>(
  thrTime: number,
  priorityStr = "priority"
): MonoTypeOperatorFunction<T> {
  return s => defer(() => {
    const priorityTimeStamp = new Map<string, number>();
    return s.pipe(
      filter(v => Date.now() - (
        priorityTimeStamp.get(v[priorityStr]) ||
        0) >= thrTime
      ),
      tap(v => {
        if(v[priorityStr] != null){
          priorityTimeStamp.set(
            v[priorityStr],
            Date.now()
          )
        }
      })
    );
  });
}
```
