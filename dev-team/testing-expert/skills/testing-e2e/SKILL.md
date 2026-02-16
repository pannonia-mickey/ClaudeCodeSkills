---
name: E2E Testing
description: This skill should be used when the user asks about "Playwright testing", "Cypress testing", "E2E test setup", "page object model", "browser automation", "visual regression testing", "accessibility testing", or "multi-browser testing". It covers Playwright and Cypress patterns for end-to-end testing.
---

# E2E Testing

## Playwright Setup

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  reporter: [['html'], ['junit', { outputFile: 'test-results/results.xml' }]],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'npm run dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

## Page Object Model

```typescript
// e2e/pages/login.page.ts
export class LoginPage {
  constructor(private page: Page) {}

  async goto() { await this.page.goto('/login'); }
  async fillEmail(email: string) { await this.page.getByLabel('Email').fill(email); }
  async fillPassword(password: string) { await this.page.getByLabel('Password').fill(password); }
  async submit() { await this.page.getByRole('button', { name: 'Sign in' }).click(); }

  async login(email: string, password: string) {
    await this.goto();
    await this.fillEmail(email);
    await this.fillPassword(password);
    await this.submit();
  }

  async expectError(message: string) {
    await expect(this.page.getByRole('alert')).toContainText(message);
  }
}

// e2e/tests/login.spec.ts
test('should login successfully', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.login('user@test.com', 'password123');
  await expect(page).toHaveURL('/dashboard');
});
```

## Test Fixtures

```typescript
// e2e/fixtures.ts
import { test as base } from '@playwright/test';

type TestFixtures = {
  authenticatedPage: Page;
  testUser: { email: string; password: string };
};

export const test = base.extend<TestFixtures>({
  testUser: async ({}, use) => {
    const user = { email: `test-${Date.now()}@test.com`, password: 'Test123!' };
    await createTestUser(user);
    await use(user);
    await deleteTestUser(user.email);
  },

  authenticatedPage: async ({ page, testUser }, use) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill(testUser.email);
    await page.getByLabel('Password').fill(testUser.password);
    await page.getByRole('button', { name: 'Sign in' }).click();
    await page.waitForURL('/dashboard');
    await use(page);
  },
});
```

## Visual Regression

```typescript
test('checkout page matches snapshot', async ({ page }) => {
  await page.goto('/checkout');
  await expect(page).toHaveScreenshot('checkout.png', {
    maxDiffPixelRatio: 0.01,
    mask: [page.locator('[data-testid="timestamp"]')], // Ignore dynamic content
  });
});
```

## Accessibility Testing

```typescript
import AxeBuilder from '@axe-core/playwright';

test('checkout page is accessible', async ({ page }) => {
  await page.goto('/checkout');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

## References

- [Playwright Patterns](references/playwright-patterns.md) — Network mocking, authentication state, API testing, mobile testing.
- [Visual Testing](references/visual-testing.md) — Screenshot comparison, Percy/Chromatic integration, handling dynamic content.
