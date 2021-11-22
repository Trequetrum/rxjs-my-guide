# Initialize

Like `tap`, but performs some side-effect immediately after subscribing.

```TypeScript
export function initialize<T>(effect: () => void): MonoTypeOperatorFunction<T> {
  return s => new Observable(ob => {
    const sub = s.subscribe(ob);
    effect();
    return sub;
  });
}
```
 Example:
 ```TypeScript
const a$ = new Subject<number>();

a$.pipe(
  initialize(() => a$.next(5)),
  map(v => v + 1)
).subscribe(console.log);
 ```
