---
name: Angular Testing
description: This skill should be used when the user asks to "test Angular component", "test Angular service", "mock Angular HTTP", "TestBed setup", "Angular integration test", "test Angular form", "test Angular router", "Playwright Angular", or "Jest Angular". It covers TestBed, component/service/HTTP testing, form testing, and E2E.
---

# Angular Testing

## TestBed Setup (Standalone Components)

```typescript
describe('ProductCardComponent', () => {
  let component: ProductCardComponent;
  let fixture: ComponentFixture<ProductCardComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ProductCardComponent], // Standalone — use imports, not declarations
    }).compileComponents();

    fixture = TestBed.createComponent(ProductCardComponent);
    component = fixture.componentInstance;
  });

  it('should display product name', () => {
    fixture.componentRef.setInput('product', { id: '1', name: 'Widget', price: 9.99 });
    fixture.detectChanges();
    const el = fixture.nativeElement.querySelector('h3');
    expect(el.textContent).toContain('Widget');
  });

  it('should emit addToCart on button click', () => {
    const product = { id: '1', name: 'Widget', price: 9.99 };
    fixture.componentRef.setInput('product', product);
    fixture.detectChanges();

    const spy = jest.fn();
    component.addToCart.subscribe(spy);

    fixture.nativeElement.querySelector('button').click();
    expect(spy).toHaveBeenCalledWith(product);
  });
});
```

## Service Testing

```typescript
describe('ProductService', () => {
  let service: ProductService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [ProductService, provideHttpClient(), provideHttpClientTesting()],
    });
    service = TestBed.inject(ProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify()); // Ensure no outstanding requests

  it('should fetch products', () => {
    const mockProducts: Product[] = [{ id: '1', name: 'Widget', price: 9.99 }];

    service.getAll().subscribe(products => {
      expect(products).toEqual(mockProducts);
    });

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('GET');
    req.flush(mockProducts);
  });

  it('should handle errors', () => {
    service.getAll().subscribe({
      error: err => expect(err.status).toBe(500),
    });

    httpMock.expectOne('/api/products').flush('Error', { status: 500, statusText: 'Server Error' });
  });
});
```

## Form Testing

```typescript
describe('LoginFormComponent', () => {
  let component: LoginFormComponent;
  let fixture: ComponentFixture<LoginFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [LoginFormComponent] }).compileComponents();
    fixture = TestBed.createComponent(LoginFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should be invalid when empty', () => {
    expect(component.form.valid).toBeFalse();
  });

  it('should be valid with correct data', () => {
    component.form.patchValue({ email: 'test@example.com', password: 'password123' });
    expect(component.form.valid).toBeTrue();
  });

  it('should show email error when invalid', () => {
    component.form.get('email')!.setValue('invalid');
    component.form.get('email')!.markAsTouched();
    fixture.detectChanges();
    const error = fixture.nativeElement.querySelector('.error-message');
    expect(error.textContent).toContain('valid email');
  });
});
```

## Testing with fakeAsync/tick

```typescript
it('should debounce search input', fakeAsync(() => {
  component.searchControl.setValue('widget');
  tick(200); // Not enough time
  expect(component.results()).toEqual([]);

  tick(100); // Total 300ms — debounce fires
  fixture.detectChanges();
  // Assert search was triggered
}));
```

## Signal Testing

```typescript
it('should update computed when signal changes', () => {
  component.searchTerm.set('widget');
  // Computed updates synchronously
  expect(component.filteredProducts().length).toBe(1);
});
```

## Router Testing

```typescript
it('should navigate to product detail', async () => {
  const harness = await RouterTestingHarness.create();
  const component = await harness.navigateByUrl('/products/1', ProductDetailComponent);
  expect(component.productId()).toBe('1');
});
```

## E2E with Playwright

```typescript
import { test, expect } from '@playwright/test';

test('should display product list', async ({ page }) => {
  await page.goto('/products');
  await expect(page.getByRole('heading', { name: 'Products' })).toBeVisible();
  await expect(page.getByTestId('product-card')).toHaveCount(10);
});

test('should search products', async ({ page }) => {
  await page.goto('/products');
  await page.getByPlaceholder('Search...').fill('widget');
  await expect(page.getByTestId('product-card')).toHaveCount(1);
});
```

## References

- [Testing Patterns](references/testing-patterns.md) — Component harness, testing observables, interceptors, guards, directives.
- [Testing Setup](references/testing-setup.md) — Jest setup, Spectator, ng-mocks, CI integration, code coverage.
