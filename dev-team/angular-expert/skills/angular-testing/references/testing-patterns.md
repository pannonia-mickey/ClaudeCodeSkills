# Angular Testing Patterns

## Component Harness

```typescript
import { ComponentHarness, HarnessPredicate } from '@angular/cdk/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';

class ProductCardHarness extends ComponentHarness {
  static hostSelector = 'app-product-card';

  static with(options: { name?: string } = {}): HarnessPredicate<ProductCardHarness> {
    return new HarnessPredicate(ProductCardHarness, options)
      .addOption('name', options.name, async (harness, name) => {
        const text = await harness.getName();
        return text === name;
      });
  }

  private getNameEl = this.locatorFor('h3');
  private getPriceEl = this.locatorFor('.price');
  private getAddButton = this.locatorFor('button.add-to-cart');

  async getName(): Promise<string> {
    return (await this.getNameEl()).text();
  }

  async getPrice(): Promise<string> {
    return (await this.getPriceEl()).text();
  }

  async addToCart(): Promise<void> {
    return (await this.getAddButton()).click();
  }
}

// Usage in tests
describe('ProductListComponent', () => {
  let loader: HarnessLoader;

  beforeEach(async () => {
    const fixture = TestBed.createComponent(ProductListComponent);
    loader = TestbedHarnessEnvironment.loader(fixture);
  });

  it('should display products', async () => {
    const cards = await loader.getAllHarnesses(ProductCardHarness);
    expect(cards.length).toBe(3);

    const firstName = await cards[0].getName();
    expect(firstName).toBe('Widget');
  });

  it('should find card by name', async () => {
    const card = await loader.getHarness(ProductCardHarness.with({ name: 'Widget' }));
    expect(await card.getPrice()).toBe('$9.99');
  });
});
```

## Testing Observables

```typescript
import { subscribeSpyTo } from '@hirez_io/observer-spy';

describe('ProductService', () => {
  it('should emit products (observer-spy)', () => {
    const result = subscribeSpyTo(service.getAll());
    httpMock.expectOne('/api/products').flush(mockProducts);

    expect(result.getLastValue()).toEqual(mockProducts);
    expect(result.receivedComplete()).toBeTrue();
  });
});

// Manual observable testing
describe('search$', () => {
  it('should debounce and search', fakeAsync(() => {
    const results: Product[][] = [];
    const sub = component.results$.subscribe(r => results.push(r));

    component.searchControl.setValue('wid');
    tick(200); // Not enough — debounce is 300ms
    expect(results.length).toBe(0);

    tick(100); // Total 300ms
    const req = httpMock.expectOne(r => r.url.includes('/api/search'));
    req.flush([mockProduct]);

    expect(results.length).toBe(1);
    expect(results[0]).toEqual([mockProduct]);

    sub.unsubscribe();
  }));
});
```

## Testing Interceptors

```typescript
describe('authInterceptor', () => {
  let httpMock: HttpTestingController;
  let http: HttpClient;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withInterceptors([authInterceptor])),
        provideHttpClientTesting(),
        { provide: AuthService, useValue: { getToken: () => 'test-token', logout: jest.fn() } },
      ],
    });
    http = TestBed.inject(HttpClient);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('should add Authorization header', () => {
    http.get('/api/data').subscribe();
    const req = httpMock.expectOne('/api/data');
    expect(req.request.headers.get('Authorization')).toBe('Bearer test-token');
  });

  it('should call logout on 401', () => {
    const authService = TestBed.inject(AuthService);
    http.get('/api/data').subscribe({ error: () => {} });
    httpMock.expectOne('/api/data').flush('Unauthorized', { status: 401, statusText: 'Unauthorized' });
    expect(authService.logout).toHaveBeenCalled();
  });
});
```

## Testing Guards

```typescript
describe('authGuard', () => {
  it('should allow authenticated users', () => {
    TestBed.configureTestingModule({
      providers: [{ provide: AuthService, useValue: { isAuthenticated: signal(true) } }],
    });

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as ActivatedRouteSnapshot, { url: '/dashboard' } as RouterStateSnapshot)
    );

    expect(result).toBeTrue();
  });

  it('should redirect unauthenticated users', () => {
    const router = TestBed.inject(Router);
    TestBed.configureTestingModule({
      providers: [
        provideRouter([]),
        { provide: AuthService, useValue: { isAuthenticated: signal(false) } },
      ],
    });

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as ActivatedRouteSnapshot, { url: '/dashboard' } as RouterStateSnapshot)
    );

    expect(result).toBeInstanceOf(UrlTree);
    expect(router.serializeUrl(result as UrlTree)).toBe('/login?returnUrl=%2Fdashboard');
  });
});
```

## Testing Directives

```typescript
@Component({
  standalone: true,
  imports: [HighlightDirective],
  template: `<p appHighlight [color]="color">Test</p>`,
})
class TestHostComponent {
  color = 'yellow';
}

describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestHostComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [TestHostComponent] }).compileComponents();
    fixture = TestBed.createComponent(TestHostComponent);
    fixture.detectChanges();
  });

  it('should highlight on mouseenter', () => {
    const p = fixture.nativeElement.querySelector('p');
    p.dispatchEvent(new Event('mouseenter'));
    fixture.detectChanges();
    expect(p.style.backgroundColor).toBe('yellow');
  });

  it('should remove highlight on mouseleave', () => {
    const p = fixture.nativeElement.querySelector('p');
    p.dispatchEvent(new Event('mouseenter'));
    p.dispatchEvent(new Event('mouseleave'));
    fixture.detectChanges();
    expect(p.style.backgroundColor).toBe('');
  });
});
```

## Testing Pipes

```typescript
describe('TruncatePipe', () => {
  const pipe = new TruncatePipe();

  it('should truncate long strings', () => {
    expect(pipe.transform('Hello World', 5)).toBe('Hello...');
  });

  it('should not truncate short strings', () => {
    expect(pipe.transform('Hi', 5)).toBe('Hi');
  });

  it('should handle null', () => {
    expect(pipe.transform(null, 5)).toBe('');
  });
});
```

## Testing with Dependency Overrides

```typescript
describe('ProductDetailComponent', () => {
  const mockProductService = {
    getById: jest.fn().mockReturnValue(of({ id: '1', name: 'Widget', price: 9.99 })),
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ProductDetailComponent],
      providers: [
        { provide: ProductService, useValue: mockProductService },
        { provide: ActivatedRoute, useValue: { params: of({ id: '1' }) } },
      ],
    }).compileComponents();
  });

  it('should load product by route param', () => {
    const fixture = TestBed.createComponent(ProductDetailComponent);
    fixture.detectChanges();
    expect(mockProductService.getById).toHaveBeenCalledWith('1');
    expect(fixture.nativeElement.querySelector('h1').textContent).toContain('Widget');
  });
});
```
