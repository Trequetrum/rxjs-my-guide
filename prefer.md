# The prefer Operator

Takes a list of observables and emits values only from the observables earlier in the list than any previous emissions.

This is useful if there are many sources for a piece of data and you want to preserve the most trusted source and use others as a fallback. 

```JavaScript
function prefer<R>(...observables: Observable<R>[]): Observable<R>{
  return new Observable(observer => {

    const subscrptions = new Array<Subscription>();
    const unsub = (index = 0) => {
      for(let i = index; i < subscrptions.length; i++){
        subscrptions[i].unsubscribe();
      }
    }

    observables
      .map(stream => publish()(stream))
      .map((stream, index) => {
        subscrptions.push(stream.subscribe((payload: R) => {
          observer.next(payload);
          unsub(index + 1);
          subscrptions.length = index + 1;
        }));
        return stream;
      })
      .forEach(stream => stream.connect());

    return { unsubscribe: () => unsub() }
  })
}
```
