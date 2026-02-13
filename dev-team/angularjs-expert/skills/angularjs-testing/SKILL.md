---
name: AngularJS Testing
description: This skill should be used when the user asks about "AngularJS test", "Jasmine AngularJS", "Karma AngularJS", "Protractor", "$httpBackend", "$componentController", "AngularJS mock", or "AngularJS e2e test". It covers AngularJS testing strategies using Jasmine and Karma for unit testing, Protractor for end-to-end testing, angular-mocks for dependency injection in tests, $httpBackend for HTTP mocking, and component/directive/service testing patterns.
---

## Testing Stack Setup

### Karma Configuration

```javascript
// karma.conf.js
module.exports = function(config) {
    config.set({
        basePath: '',
        frameworks: ['jasmine'],
        files: [
            'node_modules/angular/angular.js',
            'node_modules/angular-mocks/angular-mocks.js',
            'node_modules/angular-sanitize/angular-sanitize.js',
            'node_modules/@uirouter/angularjs/release/angular-ui-router.js',
            // Application files
            'src/app/app.module.js',
            'src/app/core/**/*.module.js',
            'src/app/core/**/*.js',
            'src/app/shared/**/*.module.js',
            'src/app/shared/**/*.js',
            'src/app/features/**/*.module.js',
            'src/app/features/**/*.js',
            // Templates
            'src/app/**/*.html',
            // Test files
            'src/app/**/*.spec.js'
        ],
        preprocessors: {
            'src/app/**/*.html': ['ng-html2js'],
            'src/app/**/!(*spec).js': ['coverage']
        },
        ngHtml2JsPreprocessor: {
            stripPrefix: 'src/app/',
            moduleName: 'templates'
        },
        reporters: ['progress', 'coverage'],
        coverageReporter: {
            type: 'html',
            dir: 'coverage/'
        },
        port: 9876,
        colors: true,
        logLevel: config.LOG_INFO,
        autoWatch: true,
        browsers: ['ChromeHeadless'],
        singleRun: false
    });
};
```

### Test File Structure

```
src/app/
    features/
        users/
            user-list.component.js
            user-list.component.spec.js     ← co-located test
            user.service.js
            user.service.spec.js            ← co-located test
    shared/
        directives/
            click-outside.directive.js
            click-outside.directive.spec.js
```

---

## Unit Testing Components

### $componentController

Use `$componentController` to instantiate a component's controller with its bindings without rendering the template.

```javascript
describe('UserListComponent', function() {
    var $componentController;
    var $rootScope;
    var UserService;
    var mockUsers;

    beforeEach(angular.mock.module('app.users'));

    beforeEach(inject(function(_$componentController_, _$rootScope_, _UserService_) {
        $componentController = _$componentController_;
        $rootScope = _$rootScope_;
        UserService = _UserService_;

        mockUsers = [
            { id: 1, name: 'Alice', email: 'alice@example.com' },
            { id: 2, name: 'Bob', email: 'bob@example.com' },
            { id: 3, name: 'Charlie', email: 'charlie@example.com' }
        ];
    }));

    it('should initialize filteredUsers from bindings', function() {
        var bindings = { users: mockUsers, onSelect: jasmine.createSpy('onSelect') };
        var ctrl = $componentController('userList', null, bindings);

        ctrl.$onInit();

        expect(ctrl.filteredUsers.length).toBe(3);
    });

    it('should filter users by search term', function() {
        var bindings = { users: mockUsers, onSelect: jasmine.createSpy('onSelect') };
        var ctrl = $componentController('userList', null, bindings);

        ctrl.$onInit();
        ctrl.searchTerm = 'alice';
        ctrl.search();

        expect(ctrl.filteredUsers.length).toBe(1);
        expect(ctrl.filteredUsers[0].name).toBe('Alice');
    });

    it('should call onSelect with selected user', function() {
        var onSelectSpy = jasmine.createSpy('onSelect');
        var bindings = { users: mockUsers, onSelect: onSelectSpy };
        var ctrl = $componentController('userList', null, bindings);

        ctrl.$onInit();
        ctrl.selectUser(mockUsers[0]);

        expect(onSelectSpy).toHaveBeenCalledWith({ $event: { user: mockUsers[0] } });
    });

    it('should update filteredUsers when users binding changes', function() {
        var bindings = { users: mockUsers, onSelect: jasmine.createSpy('onSelect') };
        var ctrl = $componentController('userList', null, bindings);
        ctrl.$onInit();

        var newUsers = [{ id: 4, name: 'Diana', email: 'diana@example.com' }];
        ctrl.users = newUsers;
        ctrl.$onChanges({
            users: { currentValue: newUsers, previousValue: mockUsers }
        });

        expect(ctrl.filteredUsers.length).toBe(1);
        expect(ctrl.filteredUsers[0].name).toBe('Diana');
    });
});
```

### Compiled Component Testing

For testing the template and bindings together:

```javascript
describe('UserList template', function() {
    var $compile, $rootScope, scope, element;

    beforeEach(angular.mock.module('app.users', 'templates'));

    beforeEach(inject(function(_$compile_, _$rootScope_) {
        $compile = _$compile_;
        $rootScope = _$rootScope_;
        scope = $rootScope.$new();
    }));

    it('should render user names', function() {
        scope.testUsers = [
            { id: 1, name: 'Alice' },
            { id: 2, name: 'Bob' }
        ];

        element = $compile(
            '<user-list users="testUsers" on-select="selected($event)"></user-list>'
        )(scope);
        scope.$digest();

        var cards = element.find('.user-card');
        expect(cards.length).toBe(2);
        expect(cards.eq(0).text()).toContain('Alice');
    });
});
```

---

## Unit Testing Services

### Testing with $httpBackend

```javascript
describe('UserService', function() {
    var UserService, $httpBackend, API_URL;

    beforeEach(angular.mock.module('app.core'));

    beforeEach(inject(function(_UserService_, _$httpBackend_, _API_URL_) {
        UserService = _UserService_;
        $httpBackend = _$httpBackend_;
        API_URL = _API_URL_;
    }));

    afterEach(function() {
        $httpBackend.verifyNoOutstandingExpectation();
        $httpBackend.verifyNoOutstandingRequest();
    });

    it('should fetch all users', function() {
        var mockResponse = [
            { id: 1, name: 'Alice' },
            { id: 2, name: 'Bob' }
        ];

        $httpBackend.expectGET(API_URL + '/users')
            .respond(200, mockResponse);

        var result;
        UserService.getAll().then(function(data) {
            result = data;
        });

        $httpBackend.flush();

        expect(result.length).toBe(2);
        expect(result[0].name).toBe('Alice');
    });

    it('should handle server errors', function() {
        $httpBackend.expectGET(API_URL + '/users')
            .respond(500, { error: 'Internal Server Error' });

        var error;
        UserService.getAll().catch(function(rejection) {
            error = rejection;
        });

        $httpBackend.flush();

        expect(error.status).toBe(500);
    });

    it('should fetch user by ID with caching', function() {
        $httpBackend.expectGET(API_URL + '/users/1')
            .respond(200, { id: 1, name: 'Alice' });

        UserService.getById(1);
        $httpBackend.flush();

        // Second call should use cache — no HTTP request expected
        var result;
        UserService.getById(1).then(function(data) {
            result = data;
        });

        $rootScope.$digest();
        expect(result.name).toBe('Alice');
    });
});
```

### Testing Interceptors

```javascript
describe('authInterceptor', function() {
    var $http, $httpBackend, TokenService;

    beforeEach(angular.mock.module('app.core'));

    beforeEach(inject(function(_$http_, _$httpBackend_, _TokenService_) {
        $http = _$http_;
        $httpBackend = _$httpBackend_;
        TokenService = _TokenService_;
    }));

    afterEach(function() {
        $httpBackend.verifyNoOutstandingExpectation();
        $httpBackend.verifyNoOutstandingRequest();
    });

    it('should attach Authorization header when token exists', function() {
        spyOn(TokenService, 'getToken').and.returnValue('test-token-123');

        $httpBackend.expectGET('/api/data', function(headers) {
            return headers.Authorization === 'Bearer test-token-123';
        }).respond(200, {});

        $http.get('/api/data');
        $httpBackend.flush();
    });

    it('should not attach header when no token', function() {
        spyOn(TokenService, 'getToken').and.returnValue(null);

        $httpBackend.expectGET('/api/data', function(headers) {
            return !headers.Authorization;
        }).respond(200, {});

        $http.get('/api/data');
        $httpBackend.flush();
    });
});
```

---

## Unit Testing Directives

```javascript
describe('clickOutside directive', function() {
    var $compile, $rootScope, $document, scope, element;

    beforeEach(angular.mock.module('app.shared'));

    beforeEach(inject(function(_$compile_, _$rootScope_, _$document_) {
        $compile = _$compile_;
        $rootScope = _$rootScope_;
        $document = _$document_;
        scope = $rootScope.$new();
    }));

    it('should call handler when clicking outside element', function() {
        scope.onOutsideClick = jasmine.createSpy('onOutsideClick');

        element = $compile(
            '<div click-outside="onOutsideClick()"><span>Inside</span></div>'
        )(scope);
        scope.$digest();

        // Simulate click outside
        var event = new Event('click', { bubbles: true });
        $document[0].body.dispatchEvent(event);
        scope.$digest();

        expect(scope.onOutsideClick).toHaveBeenCalled();
    });

    it('should not call handler when clicking inside element', function() {
        scope.onOutsideClick = jasmine.createSpy('onOutsideClick');

        element = $compile(
            '<div click-outside="onOutsideClick()"><span class="inner">Inside</span></div>'
        )(scope);
        angular.element(document.body).append(element);
        scope.$digest();

        // Simulate click inside
        element.find('span').triggerHandler('click');

        expect(scope.onOutsideClick).not.toHaveBeenCalled();

        element.remove();
    });

    it('should clean up event listener on scope destroy', function() {
        scope.onOutsideClick = jasmine.createSpy('onOutsideClick');

        element = $compile(
            '<div click-outside="onOutsideClick()">Content</div>'
        )(scope);
        scope.$digest();

        scope.$destroy();

        var event = new Event('click', { bubbles: true });
        $document[0].body.dispatchEvent(event);

        expect(scope.onOutsideClick).not.toHaveBeenCalled();
    });
});
```

---

## Unit Testing Filters

```javascript
describe('truncate filter', function() {
    var truncateFilter;

    beforeEach(angular.mock.module('app.shared'));

    beforeEach(inject(function($filter) {
        truncateFilter = $filter('truncate');
    }));

    it('should not truncate short strings', function() {
        expect(truncateFilter('Hello', 10)).toBe('Hello');
    });

    it('should truncate long strings with ellipsis', function() {
        expect(truncateFilter('Hello World', 5)).toBe('Hello...');
    });

    it('should handle null and undefined', function() {
        expect(truncateFilter(null, 10)).toBe('');
        expect(truncateFilter(undefined, 10)).toBe('');
    });
});
```

---

## End-to-End Testing with Protractor

### Configuration

```javascript
// protractor.conf.js
exports.config = {
    framework: 'jasmine',
    seleniumAddress: 'http://localhost:4444/wd/hub',
    specs: ['e2e/**/*.e2e-spec.js'],
    capabilities: {
        browserName: 'chrome',
        chromeOptions: {
            args: ['--headless', '--disable-gpu', '--no-sandbox']
        }
    },
    baseUrl: 'http://localhost:8080',
    onPrepare: function() {
        browser.waitForAngularEnabled(true);
    },
    jasmineNodeOpts: {
        defaultTimeoutInterval: 30000
    }
};
```

### Page Object Pattern

```javascript
// e2e/pages/login.page.js
var LoginPage = function() {
    this.emailInput = element(by.model('$ctrl.credentials.email'));
    this.passwordInput = element(by.model('$ctrl.credentials.password'));
    this.submitButton = element(by.css('button[type="submit"]'));
    this.errorMessage = element(by.css('.error-message'));

    this.navigate = function() {
        browser.get('/login');
    };

    this.login = function(email, password) {
        this.emailInput.clear().sendKeys(email);
        this.passwordInput.clear().sendKeys(password);
        this.submitButton.click();
    };
};

module.exports = new LoginPage();
```

```javascript
// e2e/specs/login.e2e-spec.js
var loginPage = require('../pages/login.page');

describe('Login flow', function() {
    beforeEach(function() {
        loginPage.navigate();
    });

    it('should login with valid credentials', function() {
        loginPage.login('admin@example.com', 'password123');
        expect(browser.getCurrentUrl()).toContain('/dashboard');
    });

    it('should show error for invalid credentials', function() {
        loginPage.login('wrong@example.com', 'wrongpass');
        expect(loginPage.errorMessage.isDisplayed()).toBe(true);
        expect(loginPage.errorMessage.getText()).toContain('Invalid credentials');
    });

    it('should disable submit button when form is invalid', function() {
        loginPage.emailInput.clear().sendKeys('not-an-email');
        expect(loginPage.submitButton.isEnabled()).toBe(false);
    });
});
```

---

## Test Isolation Patterns

### Module Mocking

```javascript
// Override a service with a mock for the entire test suite
beforeEach(angular.mock.module('app.users', function($provide) {
    $provide.factory('UserService', function($q) {
        return {
            getAll: jasmine.createSpy('getAll').and.returnValue(
                $q.resolve([{ id: 1, name: 'Mock User' }])
            ),
            getById: jasmine.createSpy('getById').and.returnValue(
                $q.resolve({ id: 1, name: 'Mock User' })
            )
        };
    });
}));
```

### Spy Patterns

```javascript
// Spy on existing service method
spyOn(UserService, 'getAll').and.returnValue($q.resolve(mockUsers));

// Spy and call through (observe calls while keeping real behavior)
spyOn(UserService, 'getAll').and.callThrough();

// Spy and reject (test error handling)
spyOn(UserService, 'getAll').and.returnValue($q.reject({ status: 500 }));
```

---

## References

- **[Testing Patterns](references/testing-patterns.md)** — Advanced patterns for testing route resolves, transition hooks, form validation, async operations, and cross-component integration.
- **[Testing Tools](references/testing-tools.md)** — Configuration guides for Karma plugins, code coverage, Protractor helpers, CI integration, and test reporting.
