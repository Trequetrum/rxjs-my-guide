# The log Operator

Using `tap` alongside `console.log` is pretty common while debugging, this lets a user prepend a message before logging the current value to the console.

```JavaScript
/***
 * Curried function that logs the message (msg) and
 * value (val)
 ***/
function logMsg(msg: string) {
  return (val: any = null) => 
    msg == null || msg.length < 1 ? 
      console.log(val) :
      val == null ? 
        console.log(msg) :
        console.log(`${msg}: `, val);
}

/***
 * prepend a message before logging the current value to the console.
 */
function log<T>(msg = ""): MonoTypeOperatorFunction<T> {
  return pipe(tap(logMsg(msg)));
}
```
