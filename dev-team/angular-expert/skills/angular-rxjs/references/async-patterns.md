# RxJS Async Patterns in Angular

## Retry Strategies

```typescript
// Simple retry with delay
this.http.get<Data>('/api/data').pipe(
  retry({ count: 3, delay: 1000 })
);

// Exponential backoff
this.http.get<Data>('/api/data').pipe(
  retry({
    count: 4,
    delay: (error, retryCount) => timer(Math.pow(2, retryCount) * 1000),
  })
);

// Retry only on specific errors
this.http.get<Data>('/api/data').pipe(
  retry({
    count: 3,
    delay: (error) => {
      if (error.status === 429) return timer(5000); // Rate limited
      if (error.status >= 500) return timer(1000);  // Server error
      throw error; // Don't retry client errors
    },
  })
);

// Retry with reset notifier
const retryTrigger$ = new Subject<void>();
this.http.get<Data>('/api/data').pipe(
  retry({ delay: () => retryTrigger$ })
);
// Call retryTrigger$.next() to manually retry
```

## WebSocket Patterns

```typescript
@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private socket$: WebSocketSubject<any>;

  connect(url: string): Observable<any> {
    this.socket$ = webSocket({
      url,
      openObserver: { next: () => console.log('WebSocket connected') },
      closeObserver: { next: () => console.log('WebSocket closed') },
    });

    return this.socket$.pipe(
      retry({ delay: 3000 }), // Auto-reconnect on disconnect
      share()
    );
  }

  send(message: any) {
    this.socket$.next(message);
  }

  close() {
    this.socket$.complete();
  }
}

// Typed messages with multiplexing
messages$ = this.wsService.connect('wss://api.example.com/ws');

chatMessages$ = this.messages$.pipe(
  filter(msg => msg.type === 'chat'),
  map(msg => msg.payload as ChatMessage)
);

notifications$ = this.messages$.pipe(
  filter(msg => msg.type === 'notification'),
  map(msg => msg.payload as Notification)
);
```

## Caching Patterns

```typescript
@Injectable({ providedIn: 'root' })
export class CachedProductService {
  private http = inject(HttpClient);
  private cache = new Map<string, { data: Product[]; timestamp: number }>();
  private TTL = 60_000; // 1 minute

  getProducts(category: string): Observable<Product[]> {
    const cached = this.cache.get(category);
    if (cached && Date.now() - cached.timestamp < this.TTL) {
      return of(cached.data);
    }

    return this.http.get<Product[]>(`/api/products?category=${category}`).pipe(
      tap(data => this.cache.set(category, { data, timestamp: Date.now() })),
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }

  invalidate(category?: string) {
    if (category) this.cache.delete(category);
    else this.cache.clear();
  }
}
```

```typescript
// Signal-based cache
@Injectable({ providedIn: 'root' })
export class ProductCache {
  private http = inject(HttpClient);
  private _products = signal<Product[]>([]);
  private _loaded = signal(false);

  readonly products = this._products.asReadonly();
  readonly loaded = this._loaded.asReadonly();

  load(): Observable<Product[]> {
    if (this._loaded()) return of(this._products());

    return this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        this._products.set(products);
        this._loaded.set(true);
      })
    );
  }

  refresh(): Observable<Product[]> {
    this._loaded.set(false);
    return this.load();
  }
}
```

## Optimistic Updates

```typescript
@Injectable({ providedIn: 'root' })
export class TodoStore {
  private http = inject(HttpClient);
  private _todos = signal<Todo[]>([]);
  readonly todos = this._todos.asReadonly();

  toggle(id: string) {
    // Save original state for rollback
    const original = this._todos();

    // Optimistic update
    this._todos.update(todos =>
      todos.map(t => t.id === id ? { ...t, done: !t.done } : t)
    );

    // Send to server
    const todo = this._todos().find(t => t.id === id)!;
    this.http.patch(`/api/todos/${id}`, { done: todo.done }).pipe(
      catchError(error => {
        // Rollback on failure
        this._todos.set(original);
        console.error('Failed to update todo', error);
        return EMPTY;
      })
    ).subscribe();
  }
}
```

## Pagination Pattern

```typescript
@Component({ /* ... */ })
export class PaginatedListComponent {
  private productService = inject(ProductService);

  currentPage = signal(1);
  pageSize = signal(20);

  private params$ = toObservable(computed(() => ({
    page: this.currentPage(),
    size: this.pageSize(),
  })));

  result = toSignal(
    this.params$.pipe(
      switchMap(params => this.productService.getPage(params.page, params.size))
    ),
    { initialValue: { items: [], total: 0 } }
  );

  nextPage() { this.currentPage.update(p => p + 1); }
  prevPage() { this.currentPage.update(p => Math.max(1, p - 1)); }
}
```

## Polling with Pause/Resume

```typescript
@Injectable({ providedIn: 'root' })
export class PollingService {
  private paused$ = new BehaviorSubject(false);
  private http = inject(HttpClient);

  poll<T>(url: string, intervalMs = 30000): Observable<T> {
    return this.paused$.pipe(
      switchMap(paused => paused ? EMPTY : timer(0, intervalMs)),
      switchMap(() => this.http.get<T>(url)),
      retry({ count: 3, delay: 1000 }),
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }

  pause() { this.paused$.next(true); }
  resume() { this.paused$.next(false); }
}
```

## Sequential Dependent Requests

```typescript
// Chain dependent API calls
getUserWithOrders(userId: string): Observable<UserWithOrders> {
  return this.http.get<User>(`/api/users/${userId}`).pipe(
    switchMap(user =>
      this.http.get<Order[]>(`/api/users/${userId}/orders`).pipe(
        map(orders => ({ ...user, orders }))
      )
    )
  );
}

// Parallel independent + dependent follow-up
getProductPage(id: string): Observable<ProductPageData> {
  return this.http.get<Product>(`/api/products/${id}`).pipe(
    switchMap(product =>
      forkJoin({
        reviews: this.http.get<Review[]>(`/api/products/${id}/reviews`),
        related: this.http.get<Product[]>(`/api/products/${id}/related`),
        seller: this.http.get<Seller>(`/api/sellers/${product.sellerId}`),
      }).pipe(
        map(({ reviews, related, seller }) => ({ product, reviews, related, seller }))
      )
    )
  );
}
```
