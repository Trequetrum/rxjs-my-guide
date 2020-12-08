# The filterFirst Operator

`filterFirst(n)` ignores/filters the first `n` emissions from the source stream.

```JavaScript
function filterFirst<T>(n: number): MonoTypeOperatorFunction<T>{
  return s => defer(() => {
    let count = 0;
    return s.pipe(
      filter(_ => n <= count++)
    )
  });
}
```
