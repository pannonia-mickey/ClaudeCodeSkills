# E2E Patterns

## Playwright with Vue

```ts
// e2e/user-flow.spec.ts
import { test, expect } from '@playwright/test'

test.describe('User Management', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login')
    await page.fill('[data-testid="email"]', 'admin@test.com')
    await page.fill('[data-testid="password"]', 'password')
    await page.click('[data-testid="login-btn"]')
    await page.waitForURL('/dashboard')
  })

  test('creates a new user', async ({ page }) => {
    await page.goto('/users')
    await page.click('[data-testid="create-user-btn"]')

    await page.fill('[data-testid="name"]', 'New User')
    await page.fill('[data-testid="email"]', 'new@test.com')
    await page.selectOption('[data-testid="role"]', 'editor')
    await page.click('[data-testid="submit-btn"]')

    // Wait for success toast
    await expect(page.locator('.toast-success')).toBeVisible()
    await expect(page.locator('.toast-success')).toContainText('User created')

    // Verify user appears in list
    await expect(page.locator('[data-testid="user-list"]')).toContainText('New User')
  })

  test('edits an existing user', async ({ page }) => {
    await page.goto('/users')

    await page.locator('[data-testid="user-row"]').first().click()
    await page.click('[data-testid="edit-btn"]')

    await page.fill('[data-testid="name"]', 'Updated Name')
    await page.click('[data-testid="save-btn"]')

    await expect(page.locator('.toast-success')).toContainText('User updated')
  })

  test('handles search and pagination', async ({ page }) => {
    await page.goto('/users')

    // Search
    await page.fill('[data-testid="search"]', 'Alice')
    await page.waitForResponse('**/api/users?*')

    const rows = page.locator('[data-testid="user-row"]')
    await expect(rows.first()).toContainText('Alice')

    // Clear search
    await page.fill('[data-testid="search"]', '')

    // Pagination
    await page.click('[data-testid="next-page"]')
    await expect(page.locator('[data-testid="page-indicator"]')).toContainText('Page 2')
  })
})
```

## Page Objects

```ts
// e2e/pages/LoginPage.ts
import { type Page, type Locator, expect } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly submitButton: Locator
  readonly errorMessage: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByTestId('email')
    this.passwordInput = page.getByTestId('password')
    this.submitButton = page.getByTestId('login-btn')
    this.errorMessage = page.getByTestId('error-message')
  }

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.submitButton.click()
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message)
  }
}

// e2e/pages/UsersPage.ts
export class UsersPage {
  readonly page: Page

  constructor(page: Page) {
    this.page = page
  }

  async goto() {
    await this.page.goto('/users')
  }

  async search(query: string) {
    await this.page.getByTestId('search').fill(query)
    await this.page.waitForResponse('**/api/users?*')
  }

  async getUserCount() {
    return this.page.getByTestId('user-row').count()
  }

  async clickUser(index: number) {
    await this.page.getByTestId('user-row').nth(index).click()
  }
}

// Usage in tests
test('login and navigate', async ({ page }) => {
  const loginPage = new LoginPage(page)
  const usersPage = new UsersPage(page)

  await loginPage.goto()
  await loginPage.login('admin@test.com', 'password')
  await usersPage.goto()
  expect(await usersPage.getUserCount()).toBeGreaterThan(0)
})
```

## Test Fixtures

```ts
// e2e/fixtures.ts
import { test as base } from '@playwright/test'
import { LoginPage } from './pages/LoginPage'
import { UsersPage } from './pages/UsersPage'

type Fixtures = {
  loginPage: LoginPage
  usersPage: UsersPage
  authenticatedPage: Page
}

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page))
  },

  usersPage: async ({ page }, use) => {
    await use(new UsersPage(page))
  },

  authenticatedPage: async ({ page }, use) => {
    // Auto-login
    await page.goto('/login')
    await page.fill('[data-testid="email"]', 'admin@test.com')
    await page.fill('[data-testid="password"]', 'password')
    await page.click('[data-testid="login-btn"]')
    await page.waitForURL('/dashboard')
    await use(page)
  },
})

export { expect } from '@playwright/test'

// Usage
test('authenticated test', async ({ authenticatedPage: page }) => {
  await page.goto('/users')
  // Already logged in
})
```

## Visual Regression

```ts
import { test, expect } from '@playwright/test'

test('homepage visual', async ({ page }) => {
  await page.goto('/')
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixels: 100,
  })
})

test('component states', async ({ page }) => {
  await page.goto('/components/button')

  // Capture specific element
  const button = page.getByTestId('primary-button')
  await expect(button).toHaveScreenshot('button-default.png')

  // Hover state
  await button.hover()
  await expect(button).toHaveScreenshot('button-hover.png')

  // Dark mode
  await page.emulateMedia({ colorScheme: 'dark' })
  await expect(page).toHaveScreenshot('homepage-dark.png')
})
```

## Accessibility Testing

```ts
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test('homepage accessibility', async ({ page }) => {
  await page.goto('/')

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze()

  expect(results.violations).toEqual([])
})

test('form accessibility', async ({ page }) => {
  await page.goto('/users/new')

  const results = await new AxeBuilder({ page })
    .include('[data-testid="user-form"]')
    .analyze()

  expect(results.violations).toEqual([])

  // Verify keyboard navigation
  await page.keyboard.press('Tab')
  await expect(page.getByTestId('name')).toBeFocused()

  await page.keyboard.press('Tab')
  await expect(page.getByTestId('email')).toBeFocused()
})

test('screen reader labels', async ({ page }) => {
  await page.goto('/users')

  // Check aria labels
  await expect(page.getByRole('navigation')).toBeVisible()
  await expect(page.getByRole('search')).toBeVisible()
  await expect(page.getByRole('table', { name: 'Users' })).toBeVisible()
})
```
