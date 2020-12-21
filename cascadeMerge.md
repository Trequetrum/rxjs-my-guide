# The cascadeMerge Operator

A special way to merge a list of observables. Subscribe to the first observable, when the first observable emits its first value, then merge the second observable. When the second observable emits its first value, then merge to the third Observable. Continue this pattern until all the observables are merged.

```JavaScript
function cascadeMerge<T>(...observables: Observable<T>[]): Observable<T>{
  if(observables.length < 1) return EMPTY;
  return observables[0].pipe(
    mergeMap((v,i) => {
      if(i === 0 && observables.length > 1){
        return concat(
          of(v),
          cascadeMerge(...observables.slice(1))
        )
      }
      return of(v);
    })
  );
}
```
