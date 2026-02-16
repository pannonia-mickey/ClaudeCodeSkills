# Angular Modern Features

## Signals Deep Dive

### WritableSignal
```typescript
const count = signal(0);
count.set(5);           // Replace value
count.update(c => c + 1); // Transform
count.asReadonly();     // Read-only view
```

### Computed Signals
```typescript
const firstName = signal('John');
const lastName = signal('Doe');
const fullName = computed(() => `${firstName()} ${lastName()}`);
// Automatically recomputes when firstName or lastName changes
// Memoized — only recomputes when dependencies actually change
```

### Effect
```typescript
effect((onCleanup) => {
  const timer = setInterval(() => console.log(count()), 1000);
  onCleanup(() => clearInterval(timer)); // Cleanup on re-run or destroy
});
```

### Signal Equality
```typescript
// Custom equality function to avoid unnecessary updates
const user = signal({ name: 'John', age: 30 }, {
  equal: (a, b) => a.name === b.name && a.age === b.age,
});
```

## Input Signals (Angular 17.1+)

```typescript
// Replace @Input() with input()
export class ProductCardComponent {
  product = input.required<Product>();           // Required
  showPrice = input(true);                       // Optional with default
  label = input('', { alias: 'productLabel' });  // With alias
  price = input(0, { transform: numberAttribute }); // With transform
}
```

## Model Signals (Two-Way Binding)

```typescript
// Replace @Input() + @Output() pattern
export class ToggleComponent {
  checked = model(false); // Creates input + output automatically
}

// Usage: <app-toggle [(checked)]="isEnabled" />
```

## Output Function

```typescript
// Replace @Output() EventEmitter
export class ProductCardComponent {
  addToCart = output<Product>();       // Replaces @Output() addToCart = new EventEmitter<Product>();
  remove = output<void>();
}
```

## Deferrable Views

```html
<!-- Trigger options -->
@defer (on viewport) { <app-chart /> }
@defer (on interaction) { <app-comments /> }
@defer (on idle) { <app-analytics /> }
@defer (on timer(5s)) { <app-ad /> }
@defer (on hover) { <app-tooltip /> }
@defer (when isVisible()) { <app-panel /> }

<!-- Prefetch separately from render -->
@defer (on interaction; prefetch on idle) {
  <app-heavy-component />
}

<!-- With loading/placeholder/error states -->
@defer (on viewport) {
  <app-chart [data]="chartData()" />
} @loading (minimum 200ms) {
  <app-skeleton height="300px" />
} @placeholder (minimum 100ms) {
  <div class="chart-placeholder">Chart loads on scroll</div>
} @error {
  <p>Failed to load chart component.</p>
}
```

## Zoneless Angular (Experimental)

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideExperimentalZonelessChangeDetection(),
    // ... other providers
  ],
};
// Requires all components to use signals or manually trigger change detection
```
