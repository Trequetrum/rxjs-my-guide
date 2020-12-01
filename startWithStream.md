# The startWithStream operator

A more general purpose version of startWIthDefer that lets you prepend the current stream with another stream.

```JavaScript
function startWithStream<T, R>(fn: () => Observable<R>): OperatorFunction<T, T|R> {
  return s => concat(defer(fn), s);
}
```

You can use this any time you'd use startWith or startWithDefer, just be sure to return an observable stream. 
