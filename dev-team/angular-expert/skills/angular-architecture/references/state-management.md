# Angular State Management

## Comparison Matrix

| Approach | Complexity | Best For | Boilerplate |
|----------|-----------|----------|-------------|
| Signals (service) | Low | Small-medium apps, local feature state | Minimal |
| ComponentStore | Medium | Feature-level state with complex logic | Low |
| NgRx Store | High | Large apps, shared state, time-travel | High |

## Signals-Based State

```typescript
@Injectable({ providedIn: 'root' })
export class TodoStore {
  private _todos = signal<Todo[]>([]);
  readonly todos = this._todos.asReadonly();
  readonly completed = computed(() => this._todos().filter(t => t.done));
  readonly pending = computed(() => this._todos().filter(t => !t.done));

  addTodo(title: string) {
    this._todos.update(todos => [...todos, { id: crypto.randomUUID(), title, done: false }]);
  }

  toggle(id: string) {
    this._todos.update(todos => todos.map(t => t.id === id ? { ...t, done: !t.done } : t));
  }

  remove(id: string) {
    this._todos.update(todos => todos.filter(t => t.id !== id));
  }
}
```

## NgRx Store

```typescript
// Actions
export const loadProducts = createAction('[Products] Load');
export const loadProductsSuccess = createAction('[Products] Load Success', props<{ products: Product[] }>());

// Reducer
export const productsReducer = createReducer(
  initialState,
  on(loadProducts, state => ({ ...state, loading: true })),
  on(loadProductsSuccess, (state, { products }) => ({ ...state, products, loading: false })),
);

// Effects
export const loadProducts$ = createEffect((actions$ = inject(Actions), http = inject(HttpClient)) =>
  actions$.pipe(
    ofType(loadProducts),
    switchMap(() => http.get<Product[]>('/api/products').pipe(
      map(products => loadProductsSuccess({ products })),
      catchError(() => EMPTY)
    ))
  ), { functional: true }
);

// Selectors
export const selectProducts = createSelector(selectProductsState, state => state.products);
export const selectLoading = createSelector(selectProductsState, state => state.loading);
```

## ComponentStore

```typescript
interface ProductState { products: Product[]; loading: boolean; }

@Injectable()
export class ProductStore extends ComponentStore<ProductState> {
  constructor() { super({ products: [], loading: false }); }

  readonly products$ = this.select(s => s.products);
  readonly loading$ = this.select(s => s.loading);

  readonly loadProducts = this.effect<void>(trigger$ =>
    trigger$.pipe(
      tap(() => this.patchState({ loading: true })),
      switchMap(() => inject(HttpClient).get<Product[]>('/api/products').pipe(
        tapResponse(
          products => this.patchState({ products, loading: false }),
          () => this.patchState({ loading: false })
        )
      ))
    )
  );
}
```
