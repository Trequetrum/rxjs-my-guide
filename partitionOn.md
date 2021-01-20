# The partitionOn Operator

This static operator takes an Observable and an array of predicated (functions returning true/false).
This operates very similarly to how `partition` works, only it can create any number of paritions at once.

It creates a partitioned stream for each value where a passed predicate returns true and ignores all values where none of the predicates return true.

/***
 * Create a partitioned stream for each value where a passed
 * predicate returns true
 ***/
function partitionOn<T>(
  input$: Observable<T>,
  predicates: ((v: T) => boolean)[]
): Observable<T>[] {
  const partitions = predicates.map(predicate => ({
    predicate,
    stream: new Subject<T>()
  }));

  input$.subscribe({
    next: (v: T) =>
      partitions.forEach(prt => {
        if (prt.predicate(v)) {
          prt.stream.next(v);
        }
      }),
    complete: () => partitions.forEach(prt => prt.stream.complete()),
    error: err => partitions.forEach(prt => prt.stream.error(err))
  });

  return partitions.map(prt => prt.stream.asObservable());
}
