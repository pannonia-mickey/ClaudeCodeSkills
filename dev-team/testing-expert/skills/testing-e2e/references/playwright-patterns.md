# Playwright Patterns

## Network Mocking

```typescript
test('should display products from API', async ({ page }) => {
  await page.route('/api/products', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ data: [{ id: '1', name: 'Widget', price: 9.99 }] }),
    });
  });

  await page.goto('/products');
  await expect(page.getByTestId('product-card')).toHaveCount(1);
  await expect(page.getByText('Widget')).toBeVisible();
});

// Mock API errors
test('should show error state', async ({ page }) => {
  await page.route('/api/products', (route) => route.fulfill({ status: 500 }));
  await page.goto('/products');
  await expect(page.getByText('Failed to load')).toBeVisible();
});

// Intercept and modify responses
await page.route('/api/users/me', async (route) => {
  const response = await route.fetch();
  const body = await response.json();
  body.role = 'admin'; // Override role for testing
  await route.fulfill({ response, body: JSON.stringify(body) });
});
```

## Authentication State Reuse

```typescript
// Save auth state once, reuse across tests
// global-setup.ts
async function globalSetup() {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('/login');
  await page.getByLabel('Email').fill('admin@test.com');
  await page.getByLabel('Password').fill('password');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');
  await page.context().storageState({ path: 'e2e/.auth/admin.json' });
  await browser.close();
}

// Use in tests
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'authenticated',
      use: { storageState: 'e2e/.auth/admin.json' },
      dependencies: ['setup'],
    },
  ],
});
```

## API Testing with Playwright

```typescript
test('POST /api/users creates user', async ({ request }) => {
  const response = await request.post('/api/users', {
    data: { name: 'Alice', email: 'alice@test.com' },
  });

  expect(response.ok()).toBeTruthy();
  const body = await response.json();
  expect(body.data).toMatchObject({ name: 'Alice', email: 'alice@test.com' });
  expect(body.data.id).toBeDefined();
});
```

## Mobile Testing

```typescript
test('mobile navigation works', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 667 });
  await page.goto('/');

  // Mobile hamburger menu
  await page.getByRole('button', { name: 'Menu' }).click();
  await expect(page.getByRole('navigation')).toBeVisible();
  await page.getByRole('link', { name: 'Products' }).click();
  await expect(page).toHaveURL('/products');
});

// Touch gestures
await page.touchscreen.tap(200, 300);
```

## Waiting Strategies

```typescript
// Prefer auto-waiting (built into Playwright locators)
await page.getByText('Submit').click(); // Auto-waits for element

// Explicit wait for network
await page.waitForResponse('/api/checkout');

// Wait for load state
await page.waitForLoadState('networkidle');

// Custom wait condition
await page.waitForFunction(() => document.querySelectorAll('.item').length > 5);
```

## Debugging

```typescript
// Pause execution for manual inspection
await page.pause();

// Trace viewer
// npx playwright test --trace on
// npx playwright show-trace trace.zip

// Screenshot on every step
test.afterEach(async ({ page }, testInfo) => {
  if (testInfo.status !== 'passed') {
    await page.screenshot({ path: `screenshots/${testInfo.title}.png`, fullPage: true });
  }
});
```
