# The prefer Operator

Takes a list of observables and emits values only from the observables earlier in the list than any previous emissions.

This is useful if there are many sources for a piece of data and you want to preserve the most trusted source and use others as a fallback. 

```JavaScript
function prefer<R>(...observables: Observable<R>[]): Observable<R>{
  return new Observable(observer => {

    let subscriptions: Subscription[];
    const unsub = (index = 0) => {
      for(let i = index; i < subscriptions.length; i++){
        subscriptions[i].unsubscribe();
      }
    }

    const published = observables.map(stream => publish()(stream));
    subscriptions = published.map((stream, index) => 
      stream.subscribe((payload: R) => {
         observer.next(payload);
         unsub(index + 1);
         subscriptions.length = index + 1;
       })
     );
     published.forEach(stream => stream.connect());

    return { unsubscribe: () => unsub() };
  });
}
```
