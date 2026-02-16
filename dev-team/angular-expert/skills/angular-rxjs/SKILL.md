---
name: Angular RxJS
description: This skill should be used when the user asks to "use RxJS in Angular", "compose observables", "debounce search", "switchMap vs mergeMap", "unsubscribe Angular", "async pipe", "combine observables", "handle errors in RxJS", or "bridge signals and observables". It covers RxJS operators, patterns, subscription management, and signal bridging.
---

# RxJS in Angular

## Core Concepts

```typescript
// Subject — both producer and consumer
const subject = new Subject<string>();
subject.next('hello');

// BehaviorSubject — has initial value, emits last value to new subscribers
const loading$ = new BehaviorSubject<boolean>(false);
console.log(loading$.value); // false

// ReplaySubject — replays N values to new subscribers
const events$ = new ReplaySubject<Event>(3); // Buffer last 3
```

## Essential Operators

### Transformation (switchMap vs mergeMap vs concatMap vs exhaustMap)

```typescript
// switchMap — cancels previous, only latest matters (SEARCH)
searchTerm$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.http.get<Product[]>(`/api/search?q=${term}`))
);

// mergeMap — runs all in parallel (BATCH operations)
ids$.pipe(
  mergeMap(id => this.http.get<Product>(`/api/products/${id}`), 3) // concurrency limit
);

// concatMap — queues, preserves order (SEQUENTIAL)
actions$.pipe(
  concatMap(action => this.http.post('/api/actions', action))
);

// exhaustMap — ignores new while active (FORM SUBMIT)
submitClick$.pipe(
  exhaustMap(() => this.http.post('/api/orders', this.form.value))
);
```

### Combination

```typescript
// combineLatest — latest from each, emits when any changes
combineLatest([this.products$, this.filter$, this.sort$]).pipe(
  map(([products, filter, sort]) => applyFilterAndSort(products, filter, sort))
);

// forkJoin — wait for all to complete (parallel HTTP calls)
forkJoin({
  products: this.http.get<Product[]>('/api/products'),
  categories: this.http.get<Category[]>('/api/categories'),
}).subscribe(({ products, categories }) => { /* both ready */ });

// withLatestFrom — combine with latest value from another
this.save$.pipe(
  withLatestFrom(this.currentUser$),
  switchMap(([_, user]) => this.http.post('/api/save', { userId: user.id }))
);
```

## Error Handling

```typescript
this.http.get<Product[]>('/api/products').pipe(
  retry({ count: 3, delay: 1000 }),
  catchError(error => {
    console.error('Failed to load products', error);
    return of([]); // Fallback value
  })
);

// Retry with exponential backoff
this.http.get('/api/data').pipe(
  retry({ count: 3, delay: (error, retryCount) => timer(Math.pow(2, retryCount) * 1000) })
);
```

## Subscription Management

```typescript
// takeUntilDestroyed (Angular 16+, preferred)
export class MyComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    interval(1000).pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(count => console.log(count));
  }
}

// async pipe (template, auto-subscribes/unsubscribes)
@Component({
  template: `
    @if (products$ | async; as products) {
      @for (p of products; track p.id) { <app-product [product]="p" /> }
    } @else {
      <app-spinner />
    }
  `,
})
export class ProductListComponent {
  products$ = inject(ProductService).getAll();
}
```

## Bridging Signals and Observables

```typescript
// Observable → Signal
const products = toSignal(this.productService.getAll(), { initialValue: [] });

// Signal → Observable
const searchTerm = signal('');
const searchTerm$ = toObservable(searchTerm);

// Combine: signal input + observable HTTP
const categoryId = input.required<string>();
products = toSignal(
  toObservable(this.categoryId).pipe(
    switchMap(id => this.productService.getByCategory(id))
  ),
  { initialValue: [] }
);
```

## Common Patterns

### Search with Debounce

```typescript
searchControl = new FormControl('');
results$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  filter(term => term.length >= 2),
  switchMap(term => this.searchService.search(term)),
  catchError(() => of([]))
);
```

### Polling

```typescript
data$ = timer(0, 30000).pipe(
  switchMap(() => this.http.get<Data>('/api/data')),
  retry(3),
  shareReplay(1)
);
```

## References

- [Operator Patterns](references/operator-patterns.md) — Operator decision tree, multicasting, custom operators, marble testing.
- [Async Patterns](references/async-patterns.md) — Retry strategies, WebSocket, caching, optimistic updates.
