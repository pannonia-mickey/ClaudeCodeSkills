---
name: Angular Architecture
description: This skill should be used when the user asks to "structure an Angular project", "organize Angular features", "Angular state management", "lazy load routes", "Angular interceptors", "Angular guards", "smart and dumb components", or "Angular application config". It covers project structure, routing, state management, interceptors, and application configuration.
---

# Angular Project Architecture

## Feature-Based Structure

```
src/app/
  core/                        # Singleton services, guards, interceptors
    auth/
      auth.service.ts
      auth.guard.ts
      auth.interceptor.ts
    layout/
      header.component.ts
      footer.component.ts
  shared/                      # Reusable components, directives, pipes
    components/
      button.component.ts
      modal.component.ts
    directives/
      click-outside.directive.ts
    pipes/
      truncate.pipe.ts
  features/                    # Feature-specific code
    products/
      product-list.component.ts
      product-detail.component.ts
      product.service.ts
      product.routes.ts
    orders/
      order-list.component.ts
      order.service.ts
      order.routes.ts
  app.component.ts
  app.config.ts
  app.routes.ts
```

## Application Configuration

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding(), withViewTransitions()),
    provideHttpClient(withInterceptors([authInterceptor, loggingInterceptor])),
    provideAnimationsAsync(),
    { provide: API_URL, useValue: environment.apiUrl },
  ],
};
```

## Lazy-Loaded Routes

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'products', loadComponent: () => import('./features/products/product-list.component').then(m => m.ProductListComponent) },
  { path: 'admin', loadChildren: () => import('./features/admin/admin.routes').then(m => m.adminRoutes), canActivate: [authGuard] },
  { path: '**', component: NotFoundComponent },
];

// features/admin/admin.routes.ts
export const adminRoutes: Routes = [
  { path: '', component: AdminDashboardComponent },
  { path: 'users', component: UserManagementComponent },
];
```

## Functional Guards and Interceptors

```typescript
// auth.guard.ts
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  if (authService.isAuthenticated()) return true;
  return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

// auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();
  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }
  return next(req).pipe(
    catchError(error => {
      if (error.status === 401) authService.logout();
      return throwError(() => error);
    })
  );
};
```

## Smart vs Presentational Components

```typescript
// Smart (container) — manages state, calls services
@Component({
  selector: 'app-product-page',
  standalone: true,
  imports: [ProductListComponent, ProductFilterComponent],
  template: `
    <app-product-filter (filterChange)="onFilter($event)" />
    <app-product-list [products]="filteredProducts()" (select)="onSelect($event)" />
  `,
})
export class ProductPageComponent {
  private productService = inject(ProductService);
  private filter = signal<ProductFilter>({});
  products = toSignal(this.productService.getAll(), { initialValue: [] });
  filteredProducts = computed(() => applyFilter(this.products(), this.filter()));
  onFilter(f: ProductFilter) { this.filter.set(f); }
  onSelect(p: Product) { inject(Router).navigate(['/products', p.id]); }
}

// Presentational (dumb) — pure display, inputs/outputs only
@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [ProductCardComponent],
  template: `@for (p of products(); track p.id) { <app-product-card [product]="p" (click)="select.emit(p)" /> }`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductListComponent {
  products = input.required<Product[]>();
  select = output<Product>();
}
```

## State Management with Signals

```typescript
@Injectable({ providedIn: 'root' })
export class CartStore {
  private _items = signal<CartItem[]>([]);

  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((sum, i) => sum + i.price * i.quantity, 0));
  readonly count = computed(() => this._items().reduce((sum, i) => sum + i.quantity, 0));

  addItem(product: Product) {
    this._items.update(items => {
      const existing = items.find(i => i.productId === product.id);
      if (existing) return items.map(i => i.productId === product.id ? { ...i, quantity: i.quantity + 1 } : i);
      return [...items, { productId: product.id, name: product.name, price: product.price, quantity: 1 }];
    });
  }

  removeItem(productId: string) {
    this._items.update(items => items.filter(i => i.productId !== productId));
  }
}
```

## References

- [State Management](references/state-management.md) — Signals-based state, NgRx, ComponentStore, comparison matrix.
- [Project Structure](references/project-structure.md) — Monorepo with Nx, library architecture, environment management.
