# RxJS Operator Patterns

## Operator Decision Tree

```
Need to transform each emission?
├── One-to-one → map()
├── One-to-many (flatten) → Need to cancel previous?
│   ├── Yes (search, navigation) → switchMap()
│   ├── No, run parallel → mergeMap()
│   ├── No, preserve order → concatMap()
│   └── No, ignore while busy → exhaustMap()
└── Filter → filter(), take(), first(), distinctUntilChanged()

Need to combine streams?
├── Latest from each → combineLatest()
├── Wait for all to complete → forkJoin()
├── Merge into one stream → merge()
├── Emit paired values → zip()
└── Attach latest from another → withLatestFrom()

Need timing control?
├── Delay → delay(), delayWhen()
├── Throttle → throttleTime(), throttle()
├── Debounce → debounceTime(), debounce()
├── Buffer → bufferTime(), bufferCount()
└── Sample → sampleTime(), sample()
```

## Multicasting Patterns

```typescript
// shareReplay — cache last N emissions, replay to late subscribers
readonly products$ = this.http.get<Product[]>('/api/products').pipe(
  shareReplay({ bufferSize: 1, refCount: true }) // refCount: true cleans up when no subscribers
);

// share — multicast without replay
readonly events$ = fromEvent(document, 'click').pipe(
  share()
);

// connectable — manual control over subscription
const source$ = connectable(interval(1000).pipe(map(i => i * 10)));
source$.subscribe(v => console.log('A:', v));
source$.subscribe(v => console.log('B:', v));
source$.connect(); // Both subscribers receive same values
```

## Custom Operators

```typescript
// Reusable operator for debounced search
function searchOperator<T>(searchFn: (term: string) => Observable<T[]>) {
  return (source$: Observable<string>) =>
    source$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter(term => term.length >= 2),
      switchMap(term => searchFn(term).pipe(
        catchError(() => of([] as T[]))
      ))
    );
}

// Usage
results$ = this.searchControl.valueChanges.pipe(
  searchOperator(term => this.productService.search(term))
);
```

```typescript
// Operator that adds loading state
function withLoading<T>(loading: WritableSignal<boolean>) {
  return (source$: Observable<T>) =>
    source$.pipe(
      tap(() => loading.set(true)),
      finalize(() => loading.set(false))
    );
}

// Usage
loading = signal(false);
data$ = this.trigger$.pipe(
  switchMap(() => this.http.get<Data>('/api/data').pipe(
    withLoading(this.loading)
  ))
);
```

## Marble Testing

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('search operator', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debounce and switchMap', () => {
    scheduler.run(({ cold, hot, expectObservable }) => {
      // - = 1 frame (1ms in virtual time)
      // a, b, c = emissions
      // | = complete
      // # = error
      const source =   hot('  -a--b-----c---|');
      const expected =      '  ------b--------(c|)';
      //                       300ms debounce ^^

      const result$ = source.pipe(
        debounceTime(3), // 3 frames = 3ms virtual
        switchMap(v => cold('---(v|)', { v }))
      );

      expectObservable(result$).toBe(expected, { b: 'b', c: 'c' });
    });
  });

  it('should retry on error', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source = cold('-#', {}, new Error('fail'));
      const expected =     '------(a|)';

      let attempts = 0;
      const result$ = defer(() => {
        attempts++;
        return attempts < 3 ? source : cold('-a|');
      }).pipe(retry(3));

      expectObservable(result$).toBe(expected);
    });
  });
});
```

## Higher-Order Observable Patterns

```typescript
// Switch to new observable on each emission, cancel previous
const autocomplete$ = searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => query.length < 2
    ? of([])
    : this.http.get<string[]>(`/api/suggest?q=${query}`)
  )
);

// Merge multiple streams into one
const allEvents$ = merge(
  fromEvent(button1, 'click').pipe(map(() => 'button1')),
  fromEvent(button2, 'click').pipe(map(() => 'button2')),
  this.websocket$.pipe(map(msg => msg.type))
);

// Race — first to emit wins
const timeout$ = timer(5000).pipe(map(() => 'timeout'));
const response$ = this.http.get<Data>('/api/data');
const result$ = race(response$, timeout$);

// Conditional observable
const data$ = iif(
  () => this.authService.isAuthenticated(),
  this.http.get<Data>('/api/private-data'),
  this.http.get<Data>('/api/public-data')
);
```

## Subjects: When to Use What

```typescript
// Subject — no initial value, no replay
// Use for: event buses, action streams
const click$ = new Subject<MouseEvent>();

// BehaviorSubject — has current value, replays latest
// Use for: state that always has a value
const currentUser$ = new BehaviorSubject<User | null>(null);
console.log(currentUser$.value); // null

// ReplaySubject — replays N past values
// Use for: caching recent events
const notifications$ = new ReplaySubject<Notification>(5);

// AsyncSubject — emits only the last value, only on complete
// Use for: single async results (rare)
const result$ = new AsyncSubject<Result>();
```
