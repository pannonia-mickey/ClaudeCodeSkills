# Angular Testing Setup

## Jest Configuration

```bash
# Install
npm install -D jest @types/jest jest-preset-angular ts-jest
```

```javascript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'jest-preset-angular',
  setupFilesAfterSetup: ['<rootDir>/setup-jest.ts'],
  testPathIgnorePatterns: ['/node_modules/', '/e2e/'],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  collectCoverageFrom: [
    'src/app/**/*.ts',
    '!src/app/**/*.module.ts',
    '!src/app/**/*.routes.ts',
    '!src/app/**/index.ts',
    '!src/main.ts',
  ],
};

export default config;
```

```typescript
// setup-jest.ts
import 'jest-preset-angular/setup-jest';

// Global mocks
Object.defineProperty(window, 'getComputedStyle', {
  value: () => ({ display: 'none', appearance: ['-webkit-appearance'] }),
});
```

```json
// tsconfig.spec.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/spec",
    "types": ["jest"]
  },
  "include": ["src/**/*.spec.ts", "src/**/*.d.ts"]
}
```

## Spectator

```bash
npm install -D @ngneat/spectator
```

```typescript
import { createComponentFactory, Spectator, mockProvider } from '@ngneat/spectator/jest';

describe('ProductListComponent', () => {
  let spectator: Spectator<ProductListComponent>;
  const createComponent = createComponentFactory({
    component: ProductListComponent,
    providers: [mockProvider(ProductService, { getAll: () => of(mockProducts) })],
    detectChanges: false,
  });

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should render products', () => {
    spectator.detectChanges();
    expect(spectator.queryAll('.product-card')).toHaveLength(3);
  });

  it('should emit on click', () => {
    spectator.detectChanges();
    const output = spyOn(spectator.component.select, 'emit');
    spectator.click('.product-card:first-child');
    expect(output).toHaveBeenCalled();
  });

  it('should call service on search', () => {
    const service = spectator.inject(ProductService);
    spectator.typeInElement('widget', '.search-input');
    spectator.tick(300);
    expect(service.search).toHaveBeenCalledWith('widget');
  });
});

// Service testing with Spectator
import { createServiceFactory, SpectatorService } from '@ngneat/spectator/jest';

describe('AuthService', () => {
  let spectator: SpectatorService<AuthService>;
  const createService = createServiceFactory({
    service: AuthService,
    providers: [provideHttpClient(), provideHttpClientTesting()],
  });

  beforeEach(() => {
    spectator = createService();
  });

  it('should login', () => {
    const httpMock = spectator.inject(HttpTestingController);
    spectator.service.login({ email: 'test@example.com', password: 'pass' }).subscribe();
    httpMock.expectOne('/api/auth/login').flush({ accessToken: 'token123' });
    expect(spectator.service.isAuthenticated()).toBeTrue();
  });
});
```

## ng-mocks

```bash
npm install -D ng-mocks
```

```typescript
import { MockComponent, MockDirective, MockPipe, MockProvider, MockModule } from 'ng-mocks';

describe('ProductPageComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        ProductPageComponent,
        MockComponent(ProductCardComponent),    // Auto-mock child component
        MockComponent(ProductFilterComponent),
        MockPipe(CurrencyPipe, value => `$${value}`), // Mock pipe with transform
      ],
      providers: [
        MockProvider(ProductService, {
          getAll: jest.fn().mockReturnValue(of(mockProducts)),
        }),
      ],
    }).compileComponents();
  });

  it('should pass products to child', () => {
    const fixture = TestBed.createComponent(ProductPageComponent);
    fixture.detectChanges();

    const cards = fixture.debugElement.queryAll(By.directive(ProductCardComponent));
    expect(cards.length).toBe(mockProducts.length);
    expect(cards[0].componentInstance.product).toEqual(mockProducts[0]);
  });
});
```

## CI Integration

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx ng lint
      - run: npx ng test --watch=false --code-coverage
      - run: npx ng build --configuration=production
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage/lcov.info
```

## Code Coverage Configuration

```json
// angular.json
{
  "test": {
    "options": {
      "codeCoverage": true,
      "codeCoverageExclude": [
        "src/main.ts",
        "src/**/*.module.ts",
        "src/**/*.routes.ts",
        "src/environments/**"
      ]
    }
  }
}
```

```javascript
// jest.config.ts coverage thresholds
{
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
}
```

## Testing Utilities

```typescript
// test-utils.ts — Reusable test helpers
import { ComponentFixture } from '@angular/core/testing';

export function getElement<T>(fixture: ComponentFixture<T>, selector: string): HTMLElement {
  return fixture.nativeElement.querySelector(selector);
}

export function getAllElements<T>(fixture: ComponentFixture<T>, selector: string): HTMLElement[] {
  return Array.from(fixture.nativeElement.querySelectorAll(selector));
}

export function getText<T>(fixture: ComponentFixture<T>, selector: string): string {
  return getElement(fixture, selector)?.textContent?.trim() ?? '';
}

export function click<T>(fixture: ComponentFixture<T>, selector: string): void {
  getElement(fixture, selector).click();
  fixture.detectChanges();
}

export function setInputValue<T>(fixture: ComponentFixture<T>, selector: string, value: string): void {
  const input = getElement(fixture, selector) as HTMLInputElement;
  input.value = value;
  input.dispatchEvent(new Event('input'));
  input.dispatchEvent(new Event('change'));
  fixture.detectChanges();
}
```
