---
name: Angular Mastery
description: This skill should be used when the user asks to "create an Angular component", "use Angular signals", "implement Angular control flow", "Angular dependency injection", "Angular lifecycle hooks", "Angular content projection", "standalone component", or "Angular decorators". It covers standalone components, signals, new control flow, DI, lifecycle, and template features.
---

# Angular 18+ Core Features

## Standalone Components

```typescript
@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [CurrencyPipe, RouterLink],
  template: `
    <div class="card">
      <h3>{{ product().name }}</h3>
      <p>{{ product().price | currency }}</p>
      <a [routerLink]="['/products', product().id]">View Details</a>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductCardComponent {
  product = input.required<Product>();
  addToCart = output<Product>();
}
```

## Signals

```typescript
@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [ProductCardComponent],
  template: `
    <input [value]="searchTerm()" (input)="searchTerm.set($any($event.target).value)" placeholder="Search..." />
    <p>Showing {{ filteredProducts().length }} of {{ products().length }}</p>
    @for (product of filteredProducts(); track product.id) {
      <app-product-card [product]="product" (addToCart)="handleAdd($event)" />
    } @empty {
      <p>No products found.</p>
    }
  `,
})
export class ProductListComponent {
  private productService = inject(ProductService);

  searchTerm = signal('');
  products = toSignal(this.productService.getAll(), { initialValue: [] });

  filteredProducts = computed(() => {
    const term = this.searchTerm().toLowerCase();
    return this.products().filter(p => p.name.toLowerCase().includes(term));
  });

  handleAdd(product: Product) {
    // ...
  }
}
```

### Signal API

```typescript
// Writable signal
const count = signal(0);
count.set(5);
count.update(c => c + 1);

// Computed (read-only, derived)
const doubled = computed(() => count() * 2);

// Effect (side effects on signal changes)
effect(() => {
  console.log(`Count changed to ${count()}`);
});

// Bridge with RxJS
const count$ = toObservable(count);
const data = toSignal(httpService.getData(), { initialValue: [] });
```

## New Control Flow

```html
<!-- @if / @else -->
@if (isLoading()) {
  <app-spinner />
} @else if (error()) {
  <p class="error">{{ error() }}</p>
} @else {
  <app-content [data]="data()" />
}

<!-- @for with track (required) -->
@for (item of items(); track item.id) {
  <app-item [item]="item" />
} @empty {
  <p>No items found.</p>
}

<!-- @switch -->
@switch (status()) {
  @case ('active') { <span class="badge-green">Active</span> }
  @case ('pending') { <span class="badge-yellow">Pending</span> }
  @default { <span class="badge-gray">Unknown</span> }
}

<!-- @defer for lazy loading -->
@defer (on viewport) {
  <app-heavy-chart [data]="chartData()" />
} @loading {
  <app-skeleton />
} @placeholder {
  <p>Chart will load when scrolled into view</p>
}
```

## Dependency Injection

```typescript
// providedIn: 'root' — tree-shakable singleton
@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  getAll(): Observable<Product[]> { return this.http.get<Product[]>('/api/products'); }
}

// inject() function (preferred over constructor injection)
export class ProductListComponent {
  private productService = inject(ProductService);
  private router = inject(Router);
  private destroyRef = inject(DestroyRef);
}

// InjectionToken for configuration
export const API_URL = new InjectionToken<string>('API_URL');
// In app.config.ts: { provide: API_URL, useValue: 'https://api.example.com' }
```

## Component Lifecycle

```typescript
export class MyComponent implements OnInit, OnChanges, OnDestroy, AfterViewInit {
  ngOnChanges(changes: SimpleChanges) { /* When @Input() changes */ }
  ngOnInit() { /* After first ngOnChanges — fetch data here */ }
  ngAfterViewInit() { /* DOM is ready — init JS libs here */ }
  ngOnDestroy() { /* Cleanup subscriptions, timers */ }
}

// Modern: use DestroyRef
private destroyRef = inject(DestroyRef);
ngOnInit() {
  someObservable$.pipe(takeUntilDestroyed(this.destroyRef)).subscribe();
}
```

## Content Projection

```html
<!-- Card component template -->
<div class="card">
  <ng-content select="[card-header]" />
  <ng-content />
  <ng-content select="[card-footer]" />
</div>

<!-- Usage -->
<app-card>
  <h2 card-header>Title</h2>
  <p>Body content</p>
  <button card-footer>Action</button>
</app-card>
```

## References

- [Modern Features](references/modern-features.md) — Signals deep dive, deferrable views, input signals, zoneless Angular.
- [Best Practices](references/best-practices.md) — Project structure, standalone migration, OnPush everywhere, Angular CLI.
