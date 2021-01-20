# The log Operator

Using `tap` alongside `console.log` is pretty common while debugging, this lets a user prepend a message before logging the current value to the console.

```JavaScript
function log<T>(msg = ""): MonoTypeOperatorFunction<T> {
  return s => s.pipe(
    tap(val => {
      if(msg.length > 0)
        console.log(msg + ": ", val);
      else
        console.log(val);
    })
  )
}
```
