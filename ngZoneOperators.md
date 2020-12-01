# ngZone Operators

Now that I've spent more time developing with Angular and RxJS, I don't use these operators anymore. RxJS and asnyc pipe handle change detection such that you don't need to worry about Angular’s change detection. When used to its full potential you can turn change detection off.

That being said, I’m sure there’s some fun uses for a pipeable way to enter and leave the angular zone mind-stream. Even if only for educational purposes. 

```JavaScript
constructor(private ngZone: NgZone) { }

/*****
 * If we've left the angular zone, we can use this to re-enter
 * 
 * If a third party library returns a promise/observable, we may no longer be in
 * the angular zone (This is the case for the Google API), so now we can convert such
 * observables into ones which re-enter the angular zone
 *****/
ngZoneEnterObservable<T>(input$: Observable<T>): Observable<T> {
  return new Observable<T>(observer => {
    const sub = input$.subscribe({
      next: val => this.ngZone.run(() => observer.next(val)),
      error: err => this.ngZone.run(() => observer.error(err)),
      complete: () => this.ngZone.run(() => observer.complete()),
    });
    return { unsubscribe: () => sub.unsubscribe() };
  });
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
  return new Observable<T>(observer => {
    const sub = input$.subscribe({
      next: val => this.ngZone.runOutsideAngular(() => observer.next(val)),
      error: err => this.ngZone.runOutsideAngular(() => observer.error(err)),
      complete: () => this.ngZone.runOutsideAngular(() => observer.complete()),
    });
    return { unsubscribe: () => sub.unsubscribe() };
  });
}

/*****
 * Pipeable version of ngZoneLeaveObservable
 *****/
ngZoneLeave<T>(): MonoTypeOperatorFunction<T> {
  return this.ngZoneLeaveObservable;
}
````
