# Stopwatch

### Preamble:

I figured it would be interesting to create a custom stopWatch observable. The RxJS **way** would be to implement this by switching into and out of timers/intervals.

Another interesting way to implement this is by using `setTimeout` instead. `setTimeout` should actually require a bit less memory as we're not leaning on the observable apparatus to accomplish our timing goals

How will this work? Our custom observable creates a stream that outputs the number on the stopwatch and is controlled by a separate stream (Here called `control$`). So when `control$` emits "START", the stopWatch starts, when it emits "STOP", the stopwatch stops, and when it emits "RESET" the stopwatch sets the counter back to zero. When `control$` errors or completes, the stopwatch errors or completes.

### Implemented with switchMap and Timer

The thing I don’t really like: `switchMap` manages the subscriptions to `timer` really well up until the end. When a user completes or errors the `control$` stream, we want to unsubscribe the final timer (ending the stopwatch).

We could take care of the error case with `catchError`, but there’s no built-in way to catch the completion of a stream. With my custom operator ([catchComplete](catchComplete.md)), this is easy. 

Without, you can solve it with a subject and `takeUntil` but it looks a bit clunky.

##### No [catchComplete](catchComplete.md)

```JavaScript
function createStopwatch(control$: Observable<string>, interval = 1000): Observable<number>{
  return defer(() => {
    let toggle: boolean = false;
    let count: number = 0;

    const endTicker$ = new Subject();

    const ticker = () => {
      return timer(0, interval).pipe(
        takeUntil(endTicker$),
        map(x => count++)
      )
    }

    return control$.pipe(
      tap({
        next: _ => {/*Do nothing*/},
        complete: () => {
          endTicker$.next();
          endTicker$.complete();
        },
        error: err => {
          endTicker$.next();
          endTicker$.complete();
        }
      }),
      filter(control => 
        control === "START" ||
        control === "STOP" ||
        control === "RESET"
      ),
      switchMap(control => {
        if(control === "START" && !toggle){
          toggle = true;
          return ticker();
        }else if(control === "STOP" && toggle){
          toggle = false;
          return EMPTY;
        }else if(control === "RESET"){
          count = 0;
          if(toggle){
            return ticker();
          }
        }
        return EMPTY;
      })
    );
  });
}
```

##### With [catchComplete](catchComplete.md)

```JavaScript
function createStopwatch(control$: Observable<string>, interval = 1000): Observable<number>{
  return defer(() => {
    let toggle: boolean = false;
    let count: number = 0;

    const ticker = () => {
      return timer(0, interval).pipe(
        map(x => count++)
      )
    }

    return control$.pipe(
      catchError(_ => of("END")),
      catchComplete(_ => of("END")),
      filter(control => 
        control === "START" ||
        control === "STOP" ||
        control === "RESET" ||
        control === "END"
      ),
      switchMap(control => {
        if(control === "START" && !toggle){
          toggle = true;
          return ticker();
        }else if(control === "STOP" && toggle){
          toggle = false;
          return EMPTY;
        }else if(control === "RESET"){
          count = 0;
          if(toggle){
            return ticker();
          }
        }
        return EMPTY;
      })
    );
  });
}
```

### Implemented with setTimeout

```JavaScript
function createStopwatch(control: Observable<string>, interval = 1000): Observable<number> {
  return new Observable(observer => {
    let count: number = 0;
    let tickerId: number = null;

    const clearTicker = () => {
      if(tickerId != null){
          clearTimeout(tickerId);
          tickerId = null;
        }
    }
    const setTicker = () => {
      const recursiveTicker = () => {
        tickerId = setTimeout(() => {
          observer.next(count++);
          recursiveTicker();
        }, interval);
      }
      clearTicker();
      observer.next(count++);
      recursiveTicker();
    }

    control.subscribe({
      next: input => {
        if(input === "START" && tickerId == null){
          setTicker();
        }else if(input === "STOP"){
          clearTicker();
        }else if(input === "RESET"){
          count = 0;
          if(tickerId != null){
            setTicker();
          }
        }
      },
      complete: () => {
        clearTicker();
        observer.complete();
      },
      error: err => {
        clearTicker();
        observer.error(err);
      }
    });

    return {unsubscribe: () => clearTicker()};
  });
}
```    
    
### StopWatch in Use

Here is an example of this observable being used. I manage the control stream via a subject, but it could just as easily be merged/mapped DOM events or somesuch. 

```JavaScript
const control$ = new Subject<string>();
createStopwatch(control$, 250).subscribe(console.log);

// We send a new action to our control stream every 1 second
const actions = ["START", "STOP", "START", "RESET", "START"]

zip(from(actions), interval(1000)).pipe(
  map(([x,_]) => x),
  finalize(() => {
    // After 5 seconds, unsubscribe via the control
    // If our control finishes in any way (
    // completes, errors, or is unsubscribed), our
    // sopwatch reacts by doing the same.
    control$.complete();
  })
).subscribe(x => {
  console.log(x);
  control$.next(x);
});
```

### StopWatch in Use # 2

This controls the stopwatch with `setTimeout` instead of `interval`.

```JavaScript
const control$ = new Subject<string>();
createStopwatch(control$, 250).subscribe(console.log);

// We send a new action to our control stream every 1 second
const actions = ["START", "STOP", "START", "RESET", "START"]

actions.forEach((val, index) => {
  setTimeout(() => {
    control$.next(val);
  },
  index * 1000);
})

// Unsubscribe via the control
setTimeout(() => {
  control$.complete();
}, actions.length * 1000);
```
