# The log Operator

Using `tap` alongside `console.log` is pretty common while debugging, this lets a user prepend a message before logging the current value to the console.

```JavaScript
/***
 * Returns a function that takes a value (v) and logs the message
 ***/
function logMsg(msg:string){
  return (v:any = null) => {
    if (v != null) {
      if (msg.length > 0) console.log(msg + ": ", v);
      else console.log(v);
    } else if (msg.length > 0) {
      console.log(msg);
    }
  }
}

/***
 * prepend a message before logging the current value to the console.
 */
function log<T>(msg = ""): MonoTypeOperatorFunction<T> {
  return s => s.pipe(tap(logMsg(msg)));
}
```
