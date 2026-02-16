# Angular Project Structure Patterns

## Nx Monorepo

```
my-workspace/
  apps/
    my-app/                # Application shell
      src/app/
        app.config.ts
        app.routes.ts
    my-app-e2e/            # E2E tests
  libs/
    shared/
      ui/                  # Reusable UI components
        src/lib/
          button/
          modal/
      util/                # Utility functions, pipes
      data-access/         # Shared API clients, models
    feature-products/      # Feature library
      src/lib/
        product-list.component.ts
        product.service.ts
        product.routes.ts
    feature-auth/
      src/lib/
        login.component.ts
        auth.service.ts
  nx.json
  tsconfig.base.json
```

### Nx Library Generation

```bash
npx nx generate @nx/angular:library shared-ui --directory=libs/shared/ui --standalone --routing=false
npx nx generate @nx/angular:library feature-products --directory=libs/feature-products --standalone --lazy
```

### Enforcing Module Boundaries

```json
// nx.json or .eslintrc.json
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "depConstraints": [
          { "sourceTag": "type:app", "onlyDependOnLibsWithTags": ["type:feature", "type:shared"] },
          { "sourceTag": "type:feature", "onlyDependOnLibsWithTags": ["type:shared"] },
          { "sourceTag": "type:shared", "onlyDependOnLibsWithTags": ["type:shared"] }
        ]
      }
    ]
  }
}
```

## Standalone App Structure (No Monorepo)

```
src/app/
  core/                           # Singletons, app-wide services
    auth/
      auth.service.ts
      auth.guard.ts
      auth.interceptor.ts
    error/
      error-handler.service.ts
      global-error.interceptor.ts
    layout/
      header.component.ts
      sidebar.component.ts
      footer.component.ts
  shared/                         # Reusable, stateless
    components/
      button.component.ts
      confirm-dialog.component.ts
      data-table.component.ts
    directives/
      click-outside.directive.ts
      autofocus.directive.ts
    pipes/
      truncate.pipe.ts
      relative-time.pipe.ts
    models/
      pagination.model.ts
      api-response.model.ts
    validators/
      custom-validators.ts
  features/                       # Feature-specific, lazy-loaded
    products/
      components/
        product-list.component.ts
        product-detail.component.ts
        product-card.component.ts
      services/
        product.service.ts
        product-store.service.ts
      models/
        product.model.ts
      product.routes.ts
      index.ts                    # Barrel export
    orders/
      components/
      services/
      models/
      order.routes.ts
      index.ts
  app.component.ts
  app.config.ts
  app.routes.ts
```

## Environment Management

```typescript
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  featureFlags: {
    newDashboard: true,
    betaFeatures: true,
  },
};

// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com',
  featureFlags: {
    newDashboard: true,
    betaFeatures: false,
  },
};

// environment.staging.ts
export const environment = {
  production: true,
  apiUrl: 'https://staging-api.example.com',
  featureFlags: {
    newDashboard: true,
    betaFeatures: true,
  },
};
```

### angular.json Configuration

```json
{
  "configurations": {
    "production": {
      "fileReplacements": [
        { "replace": "src/environments/environment.ts", "with": "src/environments/environment.prod.ts" }
      ],
      "budgets": [
        { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" },
        { "type": "anyComponentStyle", "maximumWarning": "2kB", "maximumError": "4kB" }
      ]
    },
    "staging": {
      "fileReplacements": [
        { "replace": "src/environments/environment.ts", "with": "src/environments/environment.staging.ts" }
      ]
    }
  }
}
```

## InjectionToken for Configuration

```typescript
// tokens.ts
export const API_URL = new InjectionToken<string>('API_URL');
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: API_URL, useValue: environment.apiUrl },
    { provide: APP_CONFIG, useFactory: () => loadConfig() },
  ],
};

// usage in service
@Injectable({ providedIn: 'root' })
export class ProductService {
  private apiUrl = inject(API_URL);
  private http = inject(HttpClient);

  getAll() {
    return this.http.get<Product[]>(`${this.apiUrl}/products`);
  }
}
```

## Feature Module Pattern (Standalone)

```typescript
// features/products/product.routes.ts
export const productRoutes: Routes = [
  {
    path: '',
    providers: [ProductStore],  // Scoped provider
    children: [
      { path: '', component: ProductListComponent },
      { path: ':id', component: ProductDetailComponent },
      { path: ':id/edit', component: ProductEditComponent, canActivate: [authGuard] },
    ],
  },
];

// app.routes.ts
export const routes: Routes = [
  { path: 'products', loadChildren: () => import('./features/products/product.routes').then(m => m.productRoutes) },
];
```
