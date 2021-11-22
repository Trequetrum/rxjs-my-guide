# The delayWhile Operator

If the given function returns true for a value, buffer that value. If the given function returns false, then emit all buffered values in the same order the source emitted them and finaly emit the current value. 

This might be useful if a condition must be met before the next transformation can be done. Say a script must be loaded before it can be used. It might also be the case that the stream itself must meet a certain condition in order to continue. 

```JavaScript
function delayWhile<T>(fn: (T)=>boolean): MonoTypeOperatorFunction<T>{
  return s => new Observable<T>(observer => {
    const buffer = new Array<T>();
    const sub = s.subscribe({
      next: (v:T) => {
        if(fn(v)){
          buffer.push(v);
        }else{
          if(buffer.length > 0){
            buffer.forEach(observer.next.bind(observer));
            buffer.length = 0;
          }
          observer.next(v);
        }
      },
      complete: observer.complete.bind(observer),
      error: observer.error.bind(observer)
    })
    return sub;
  });
}
```

