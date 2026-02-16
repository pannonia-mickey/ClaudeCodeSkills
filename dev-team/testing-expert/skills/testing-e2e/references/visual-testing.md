# Visual Regression Testing

## Playwright Screenshot Comparison

```typescript
// Full page screenshot
test('homepage visual', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png', {
    fullPage: true,
    maxDiffPixelRatio: 0.01,
  });
});

// Component screenshot
test('navigation visual', async ({ page }) => {
  await page.goto('/');
  const nav = page.getByRole('navigation');
  await expect(nav).toHaveScreenshot('navigation.png');
});

// Handling dynamic content
test('dashboard visual', async ({ page }) => {
  await page.goto('/dashboard');

  // Mask dynamic elements
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [
      page.locator('[data-testid="timestamp"]'),
      page.locator('[data-testid="user-avatar"]'),
      page.locator('.animate'),
    ],
  });
});

// Update snapshots: npx playwright test --update-snapshots
```

## Responsive Visual Testing

```typescript
const viewports = [
  { width: 1920, height: 1080, name: 'desktop' },
  { width: 768, height: 1024, name: 'tablet' },
  { width: 375, height: 667, name: 'mobile' },
];

for (const vp of viewports) {
  test(`homepage - ${vp.name}`, async ({ page }) => {
    await page.setViewportSize({ width: vp.width, height: vp.height });
    await page.goto('/');
    await expect(page).toHaveScreenshot(`homepage-${vp.name}.png`);
  });
}
```

## Dark Mode Testing

```typescript
test('supports dark mode', async ({ page }) => {
  await page.emulateMedia({ colorScheme: 'dark' });
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage-dark.png');

  await page.emulateMedia({ colorScheme: 'light' });
  await expect(page).toHaveScreenshot('homepage-light.png');
});
```

## Strategies for Stability

```typescript
// Wait for animations to complete
await page.goto('/');
await page.waitForFunction(() => {
  const animations = document.getAnimations();
  return animations.every(a => a.playState === 'finished');
});

// Disable animations for visual tests
await page.addStyleTag({
  content: `*, *::before, *::after {
    animation-duration: 0s !important;
    transition-duration: 0s !important;
  }`,
});

// Use consistent fonts
await page.addStyleTag({
  content: `* { font-family: 'Arial', sans-serif !important; }`,
});

// Freeze time-dependent content
await page.evaluate(() => {
  Date.now = () => new Date('2025-01-15T12:00:00Z').getTime();
});
```
