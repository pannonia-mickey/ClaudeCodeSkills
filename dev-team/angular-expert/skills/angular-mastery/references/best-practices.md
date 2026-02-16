# Angular Best Practices

## Project Structure

```
src/app/
  core/          # Singleton services, guards, interceptors — imported once in app.config.ts
  shared/        # Reusable standalone components, directives, pipes
  features/      # Feature-specific code, lazy-loaded
  app.config.ts  # Application providers
  app.routes.ts  # Top-level routes
```

## Standalone Migration

1. Add `standalone: true` to components/directives/pipes
2. Move `declarations` items to `imports` in the component itself
3. Remove from NgModule declarations
4. Use `provideRouter()` instead of `RouterModule.forRoot()`
5. Use `provideHttpClient()` instead of `HttpClientModule`
6. Bootstrap with `bootstrapApplication(AppComponent, appConfig)` instead of `platformBrowserDynamic().bootstrapModule(AppModule)`

## OnPush Change Detection

Use `ChangeDetectionStrategy.OnPush` on ALL components. Angular only checks when:
- Input references change (use immutable data)
- Events fire within the component
- Async pipe emits
- Signal values change
- `markForCheck()` is called explicitly

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
```

## Environment Configuration

```typescript
// environment.ts
export const environment = { production: false, apiUrl: 'http://localhost:3000/api' };

// environment.prod.ts
export const environment = { production: true, apiUrl: 'https://api.example.com' };

// angular.json — file replacement
"configurations": {
  "production": {
    "fileReplacements": [{ "replace": "src/environments/environment.ts", "with": "src/environments/environment.prod.ts" }]
  }
}
```

## Angular CLI Configuration

```bash
ng new my-app --strict --standalone --style=scss --routing --ssr=false
# --strict: strict TypeScript + Angular template checking
# --standalone: no NgModules
```

## Barrel Exports

```typescript
// features/products/index.ts
export { ProductListComponent } from './product-list.component';
export { ProductDetailComponent } from './product-detail.component';
export { ProductService } from './product.service';
export { productRoutes } from './product.routes';
```
