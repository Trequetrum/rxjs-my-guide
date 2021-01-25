# ngZone Operators

Now that I've spent more time developing with Angular and RxJS, I don't use these operators anymore. RxJS and asnyc pipe handle change detection such that you don't need to worry about Angular’s change detection. When used to its full potential you can turn change detection off.

That being said, I’m sure there’s some fun uses for a pipeable way to enter and leave the angular zone mind-stream. Even if only for educational purposes. 

```JavaScript
constructor(private ngZone: NgZone) { }

/*****
 * Wrap every emission of an observable (next, complete, & error)
 * With a callback function, effectively removing the invocation
 * of these emissions from the observable to the callback.
 *
 * This isn't really too useful except for as a helper function for
 * our NgZone Operators where we leverage this wrapper to run an
 * observable in a specific JavaScript environment.
 *****/
callBackWrapperObservable<T>(
  input$: Observable<T>, 
  callback: (fn: (v) => void) => void
): Observable<T> {
  const callBackBind = fn => (v = undefined) => callback(() => fn(v))
  return new Observable<T>(observer => {
    const sub = input$.subscribe({
      next: callBackBind(observer.next.bind(observer)),
      error: callBackBind(observer.error.bind(observer)),
      complete: callBackBind(observer.complete.bind(observer))
    });
    return { unsubscribe: () => sub.unsubscribe() };
  });
}

/*****
 * If we've left the angular zone, we can use this to re-enter
 * 
 * If a third party library returns a promise/observable, we may no longer be in
 * the angular zone (This is the case for the Google API), so now we can convert such
 * observables into ones which re-enter the angular zone
 *****/
ngZoneEnterObservable<T>(input$: Observable<T>): Observable<T> {
  return callBackWrapperObservable(input$, this.ngZone.run.bind(this.ngZone));
}

/*****
 * This is a pipeable version of ngZoneEnterObservable
 *****/
ngZoneEnter<T>(): MonoTypeOperatorFunction<T> {
  return this.ngZoneEnterObservable;
}

/*****
 * Any actions performed on the output of this observable will not trigger 
 * angular change detection. 
 *****/
ngZoneLeaveObservable<T>(input$: Observable<T>): Observable<T> {
  return callBackWrapperObservable(input$, this.ngZone.runOutsideAngular.bind(this.ngZone));
}

/*****
 * Pipeable version of ngZoneLeaveObservable
 *****/
ngZoneLeave<T>(): MonoTypeOperatorFunction<T> {
  return this.ngZoneLeaveObservable;
}
````
