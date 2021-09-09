# Stopwatch

### Preamble:

I figured it would be interesting to create a custom stopWatch observable. The RxJS **way** would be to implement this by switching into and out of timers/intervals.

Another interesting way to implement this is by using `setTimeout` instead. `setTimeout` should actually require a bit less memory as we're not leaning on the observable apparatus to accomplish our timing goals

How will this work? Our custom observable creates a stream that outputs the number on the stopwatch and is controlled by a separate stream (Here called `control$`). So when `control$` emits "START", the stopWatch starts, when it emits "STOP", the stopwatch stops, and when it emits "RESET" the stopwatch sets the counter back to zero. When `control$` errors or completes, the stopwatch errors or completes.

### Implemented with switchMap and Timer

```TypeScript
type StopwatchAction = "START" | "STOP" | "RESET" | "END";

function createStopwatch(
  control$: Observable<StopwatchAction>, 
  interval = 1000
): Observable<number>{

  return defer(() => {
    let toggle: boolean = false;
    let count: number = 0;

    const ticker = timer(0, interval).pipe(
      map(x => count++)
    );
    const end$ = of("END");

    return concat(
      control$,
      end$
    ).pipe(
      catchError(_ => end$),
      switchMap(control => {
        if(control === "START" && !toggle){
          toggle = true;
          return ticker;
        }else if(control === "STOP" && toggle){
          toggle = false;
          return EMPTY;
        }else if(control === "RESET"){
          count = 0;
          if(toggle){
            return ticker;
          }
        }
        return EMPTY;
      })
    );
  });
}
```

### Implemented with setTimeout

```TypeScript
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

### Create a stopwatch as an object

If the control stream is going to be a subject, this is a good way to create the stopwatch.

```TypeScript
function getStopWatch(interval: number = 1000): {
  control$: Subject<StopwatchAction>, 
  display$: Observable<number>
} {
  const control$ = new Subject<StopwatchAction>();
  return {
    control$,
    display$: createStopwatch(control$, interval)
  }
}
```

Stopwatch Object in Use:
```TypeScript
const watch = getStopWatch();
watch.display$.subscribe(/*Numbers emitted here every interval once started by control$*/);
watch.control$.next("START");
watch.control$.next("STOP");
watch.control$.next("RESET");
// Completing the control cleans up everything
watch.control$.complete();
```

### StopWatch in Use

Here is an example of this observable being used. I manage the control stream via a subject, but it could just as easily be merged/mapped DOM events or somesuch. 

```TypeScript
const watch = getStopWatch(250);
watch.display$.subscribe(console.log);

// We send a new action to our control stream every 1 second
const actions: StopwatchAction[] = ["START", "STOP", "START", "RESET", "START"]

zip(from(actions), interval(1000)).pipe(
  map(([x,_]) => x),
  finalize(() => {
    // After 5 seconds, unsubscribe via the control
    // If our control finishes in any way (
    // completes, errors, or is unsubscribed), our
    // sopwatch reacts by doing the same.
    watch.control$.complete();
  })
).subscribe(action => {
  console.log(action);
  watch.control$.next(action);
});
```

### StopWatch in Use # 2

This controls the stopwatch with `setTimeout` instead of `interval`.

```TypeScript
const watch = getStopWatch(250);
watch.display$.subscribe(console.log);

// We send a new action to our control stream every 1 second
const actions: StopwatchAction[] = ["START", "STOP", "START", "RESET", "START"]

actions.forEach((action, index) => {
  setTimeout(() => {
    console.log(action);
    watch.control$.next(action);
  },
  index * 1000);
})

// Unsubscribe via the control
setTimeout(() => {
  watch.control$.complete();
}, actions.length * 1000);
```

### StopWatch in Use # 3

Control a stopwatch with DOM events to set fields on the DOM.

```TypeScript
createStopwatch(merge(
  fromEvent(startBtn, 'click').pipe(mapTo("START")),
  fromEvent(resetBtn, 'click').pipe(mapTo("RESET"))
)).subscribe(seconds => {
  secondsField.innerHTML  = seconds % 60;
  minuitesField.innerHTML = Math.floor(seconds / 60) % 60;
  hoursField.innerHTML    = Math.floor(seconds / 3600);
});
```
