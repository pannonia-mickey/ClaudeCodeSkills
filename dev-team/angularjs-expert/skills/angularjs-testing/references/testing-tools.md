# AngularJS Testing Tools

Configuration guides for Karma plugins, code coverage, Protractor helpers, CI integration, and test reporting.

---

## Karma Plugins

### karma-coverage (Istanbul)

```javascript
// karma.conf.js additions
preprocessors: {
    'src/app/**/!(*spec|*mock).js': ['coverage']
},
reporters: ['progress', 'coverage'],
coverageReporter: {
    reporters: [
        { type: 'html', dir: 'coverage/', subdir: 'html' },
        { type: 'lcov', dir: 'coverage/', subdir: 'lcov' },
        { type: 'text-summary' }
    ],
    check: {
        global: {
            statements: 80,
            branches: 75,
            functions: 80,
            lines: 80
        }
    }
}
```

### karma-ng-html2js-preprocessor

Pre-compile templates for component tests:

```javascript
preprocessors: {
    'src/app/**/*.html': ['ng-html2js']
},
ngHtml2JsPreprocessor: {
    stripPrefix: 'src/app/',
    moduleName: 'templates'
}
```

In tests, load the templates module:

```javascript
beforeEach(angular.mock.module('app.users', 'templates'));
```

### karma-jasmine-html-reporter

```javascript
reporters: ['progress', 'kjhtml'],
client: {
    clearContext: false
}
```

### karma-spec-reporter

```javascript
reporters: ['spec'],
specReporter: {
    maxLogLines: 5,
    suppressErrorSummary: false,
    suppressFailed: false,
    suppressPassed: false,
    suppressSkipped: true,
    showSpecTiming: true
}
```

---

## Protractor Helpers

### Wait Conditions

```javascript
var EC = protractor.ExpectedConditions;

// Wait for element to be visible
browser.wait(EC.visibilityOf(element(by.css('.modal'))), 5000);

// Wait for element to be clickable
browser.wait(EC.elementToBeClickable(element(by.id('submit'))), 5000);

// Wait for URL to contain a string
browser.wait(EC.urlContains('/dashboard'), 5000);

// Wait for text to appear
browser.wait(EC.textToBePresentInElement(element(by.css('.status')), 'Loaded'), 5000);

// Custom wait condition
browser.wait(function() {
    return element.all(by.repeater('item in $ctrl.items')).count()
        .then(function(count) {
            return count > 0;
        });
}, 5000, 'Items did not load within 5 seconds');
```

### Common Locator Strategies

| Locator | Use Case | Example |
|---------|----------|---------|
| `by.model('$ctrl.name')` | Form inputs with ng-model | `element(by.model('$ctrl.email'))` |
| `by.binding('$ctrl.total')` | Text bound with `{{ }}` or `ng-bind` | `element(by.binding('$ctrl.total'))` |
| `by.repeater('item in $ctrl.items')` | ng-repeat rows | `element.all(by.repeater('item in $ctrl.items'))` |
| `by.css('.class-name')` | CSS selector | `element(by.css('.submit-btn'))` |
| `by.cssContainingText('.class', 'text')` | CSS + text content | `element(by.cssContainingText('h1', 'Dashboard'))` |

### Repeater Helpers

```javascript
// Get all rows
var rows = element.all(by.repeater('user in $ctrl.users'));

// Get specific row
var firstRow = rows.get(0);

// Get column from row
var name = element(by.repeater('user in $ctrl.users').row(0).column('user.name'));

// Count rows
expect(rows.count()).toBe(10);
```

---

## CI Integration

### GitHub Actions

```yaml
name: AngularJS Tests
on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: 'npm'
      - run: npm ci
      - run: npm run test:ci
      - uses: codecov/codecov-action@v4
        with:
          file: ./coverage/lcov/lcov.info

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18
      - run: npm ci
      - run: npm start &
      - run: npx wait-on http://localhost:8080
      - run: npm run e2e:ci
```

### Package Scripts

```json
{
    "scripts": {
        "test": "karma start karma.conf.js",
        "test:ci": "karma start karma.conf.js --single-run --browsers ChromeHeadless",
        "test:watch": "karma start karma.conf.js --auto-watch",
        "test:coverage": "karma start karma.conf.js --single-run --reporters coverage",
        "e2e": "protractor protractor.conf.js",
        "e2e:ci": "protractor protractor.conf.js --capabilities.chromeOptions.args=--headless"
    }
}
```

---

## angular-mocks Reference

### Core Mock Services

| Mock | Replaces | Purpose |
|------|----------|---------|
| `$httpBackend` | `$http` | Intercept and respond to HTTP requests |
| `$timeout.flush()` | `$timeout` | Synchronously flush pending timeouts |
| `$interval.flush()` | `$interval` | Synchronously flush pending intervals |
| `$log` | `$log` | Capture log messages without console output |
| `$exceptionHandler` | `$exceptionHandler` | Capture exceptions for assertion |

### $httpBackend Methods

```javascript
// Expect a specific request (order matters, fails if not called)
$httpBackend.expectGET('/api/users').respond(200, []);
$httpBackend.expectPOST('/api/users', { name: 'Alice' }).respond(201, { id: 1 });

// When — respond to any matching request (order does not matter)
$httpBackend.whenGET('/api/config').respond(200, { version: '1.0' });

// Respond with headers
$httpBackend.expectGET('/api/data').respond(200, [], { 'X-Total-Count': '100' });

// Respond with specific status
$httpBackend.expectGET('/api/data').respond(404, { error: 'Not found' });

// Flush all pending requests
$httpBackend.flush();

// Flush a specific number of requests
$httpBackend.flush(1);

// Verify all expected requests were made
$httpBackend.verifyNoOutstandingExpectation();

// Verify no unhandled requests remain
$httpBackend.verifyNoOutstandingRequest();
```
