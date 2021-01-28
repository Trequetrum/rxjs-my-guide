# Understanding `concatMap`, `mergeMap`, & `switchMap`

Some simple code that exemplifies the difference between these three higher-order observable operators. 

## The Setup 

We'll have 2 helper functions. They both use `intervalArray` (a customer operator, [linked here](intervalArray.md))

```JavaScript
/****
 * Emit 1, 2, 3, then complete: each 0.5 seconds apart
 ****/
function numberStream(): Observable<number> {
  return of([1,2,3]).pipe(
    intervalArray(500)
  );
}

/****
 * maps:
 *   1 => 10, 11, 12, then complete: each 1 second apart 
 *   2 => 20, 21, 22, then complete: each 1 second apart
 *   3 => 30, 31, 32, then complete: each 1 second apart
 ****/
function numberToStream(num): Observable<number>{
  return of([num*10, num*10+1, num*10+2]).pipe(
    intervalArray(1000)
  );
}
```

The above mapping function (`numberToStream`), takes care of the **map** part of `concatMap`, `mergeMap`, and `switchMap`

## Subscribing to each operator 

The following three snippits of code will all have different outputs:

```JavaScript
numberStream().pipe(
  concatMap(numberToStream)
).subscribe(console.log);
```
```JavaScript
numberStream().pipe(
  mergeMap(numberToStream)
).subscribe(console.log);
```
```JavaScript
numberStream().pipe(
  switchMap(numberToStream)
).subscribe(console.log);
```

### concatMap:

`concatMap` will not subscribe to the second inner observable until the first one is complete. That means that the number 13 will be emitted before the second observable (starting with the number 20) will be subscribed to.

The output:
```HTML
10 11 12 20 21 22 30 31 32
```

All the 10s are before the 20s and all the 20s are before the 30s

### mergeMap:

`mergeMap` will subscribe to the second observable the moment the second value arrives and then to the third observable the moment the third value arrives. It doesn't care about the order of output or anything like that.

The output
```HTML
10 20 11 30 21 12 31 22 32
```
The 10s are earlier because they started earlier and the 30s are later because they start later, but there's some interleaving in the middle.

### switchMap

`switchMap` will subscribe to the first observable the moment the first value arrives. It will unsubscribe to the first observable and subscribe to the second observable the moment the second value arrives (and so on).

The output
```HTML
10 20 30 31 32
```

Only the final observable ran to completion in this case. The first two only had time to emit their first value before being unsubscribed. Just like concatMap, there is no interleaving and only one inner observable is running at a time, but some emissions are effectively dropped.

## Just the code, so you can tinker:

```JavaScript
/****
 * Pipeable Operator:
 * Takes arrays emitted by the source and spaces out their
 * values by the given interval time in milliseconds
 ****/
function intervalArray<T>(intervalTime = 1000): OperatorFunction<T[], T> {
  return s => s.pipe(
    concatMap((v: T[]) => concat(
      ...v.map((value: T) => EMPTY.pipe(
        delay(intervalTime),
        startWith(value)
      ))
    ))
  );
}

/****
 * Emit 1, 2, 3, then complete: each 0.5 seconds apart
 ****/
function numberStream(): Observable<number> {
  return of([1,2,3]).pipe(
    intervalArray(500)
  );
}

/****
 * maps:
 *   1 => 10, 11, 12, then complete: each 1 second apart 
 *   2 => 20, 21, 22, then complete: each 1 second apart
 *   3 => 30, 31, 32, then complete: each 1 second apart
 ****/
function numberToStream(num): Observable<number>{
  return of([num*10, num*10+1, num*10+2]).pipe(
    intervalArray(1000)
  );
}

/****
 * Run all three operators back-to-back
 ****
concat(
  ...[concatMap, mergeMap, switchMap].map(
    op => numberStream().pipe(
      op(numberToStream),
      startWith(`${op.name}: `)
    )
  )
).subscribe(console.log);
```
