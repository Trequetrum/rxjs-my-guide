#The startWithDefer Operator

This is a version of startWith that generates the first value at the time of subscription rather than the time of creation. This is useful if you want each new subscriber to get a value based on the (possibly changing) state of the program. 

```JavaScript
function startWithDefer<T, R>(fn: () => R): OperatorFunction<T, T | R> {
	return input$ => defer(() => input$.pipe(startWith(fn())));
}
```

### `startWithDefer` in use:

A stream of the current username and future updates of that name. Since we don't know when the caller will subscribe (or if they'll subscribe multiple times), we `defer` grabbing the current username until the observable is subscribed. 

```JavaScript
function getUserName(): Observable<string> {
	return this.userNameUpdates$.pipe(
    startWithDefer(() => this.currentUserName)
  );
}
```
