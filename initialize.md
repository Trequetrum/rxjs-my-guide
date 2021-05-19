# Initialize

Like `tap`, but performs some side-effect immediately after subscribing.

```TypeScript
export function initialize<T>(fn: () => void): MonoTypeOperatorFunction<T> {
  return s => new Observable(observer => {
    const bindOn = name => observer[name].bind(observer);
    const sub = s.subscribe({
      next: bindOn("next"),
      error: bindOn("error"),
      complete: bindOn("complete")
    });
    fn();
    return {
      unsubscribe: () => sub.unsubscribe()
    };
  });
}
```
