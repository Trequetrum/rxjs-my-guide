# The priorityDebounceTime Operator

The source observable must emit values with the following shape
(Note that the key '`priority`' can be changed from its default).

```
{
  ...payload,
  priority: string
}
```

This operator functions the same way as `debounceTime`, but treats distinct priorities as separate streams. This means only values with the same priority will debounce each other.

``` JavaScript
function priorityDebounceTime<T>(
  dbTime: number,
  priorityStr = "priority"
): MonoTypeOperatorFunction<T> {
  return s => defer(() => {
    const priorityTimeStamp = new Map<string, number>();
    return s.pipe(
      mergeMap(v => {
        priorityTimeStamp.set(v[priorityStr], Date.now());
        return timer(dbTime).pipe(
          timestamp(),
          filter(({ timestamp }) =>
            timestamp - priorityTimeStamp.get(v[priorityStr]) >= dbTime
          ),
          mapTo(v)
        );
      })
    );
  });
}
```
