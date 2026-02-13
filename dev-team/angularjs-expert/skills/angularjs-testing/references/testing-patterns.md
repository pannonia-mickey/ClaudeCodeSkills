# AngularJS Testing Patterns

Advanced reference covering testing patterns for route resolves, transition hooks, form validation, async operations, and cross-component integration.

---

## Testing Route Resolves

```javascript
describe('Users route resolves', function() {
    var $state, $rootScope, $httpBackend, API_URL;

    beforeEach(angular.mock.module('app.users'));

    beforeEach(inject(function(_$state_, _$rootScope_, _$httpBackend_, _API_URL_) {
        $state = _$state_;
        $rootScope = _$rootScope_;
        $httpBackend = _$httpBackend_;
        API_URL = _API_URL_;
    }));

    afterEach(function() {
        $httpBackend.verifyNoOutstandingExpectation();
        $httpBackend.verifyNoOutstandingRequest();
    });

    it('should resolve users list before entering state', function() {
        var mockUsers = [{ id: 1, name: 'Alice' }];
        $httpBackend.expectGET(API_URL + '/users').respond(200, mockUsers);

        $state.go('users');
        $httpBackend.flush();
        $rootScope.$digest();

        expect($state.current.name).toBe('users');
    });

    it('should not enter state when resolve fails', function() {
        $httpBackend.expectGET(API_URL + '/users').respond(500);

        $state.go('users');
        $httpBackend.flush();
        $rootScope.$digest();

        expect($state.current.name).not.toBe('users');
    });
});
```

---

## Testing Transition Hooks

```javascript
describe('Auth guard', function() {
    var $state, $rootScope, AuthService;

    beforeEach(angular.mock.module('app', function($provide) {
        $provide.factory('AuthService', function() {
            return {
                isAuthenticated: jasmine.createSpy('isAuthenticated'),
                getCurrentUser: jasmine.createSpy('getCurrentUser')
            };
        });
    }));

    beforeEach(inject(function(_$state_, _$rootScope_, _AuthService_) {
        $state = _$state_;
        $rootScope = _$rootScope_;
        AuthService = _AuthService_;
    }));

    it('should redirect to login when not authenticated', function() {
        AuthService.isAuthenticated.and.returnValue(false);

        $state.go('admin');
        $rootScope.$digest();

        expect($state.current.name).toBe('login');
    });

    it('should allow access when authenticated', function() {
        AuthService.isAuthenticated.and.returnValue(true);
        AuthService.getCurrentUser.and.returnValue(
            inject(function($q) { return $q.resolve({ roles: ['admin'] }); })
        );

        $state.go('admin');
        $rootScope.$digest();

        expect($state.current.name).toBe('admin');
    });
});
```

---

## Testing Form Validation

```javascript
describe('UserForm validation', function() {
    var $compile, $rootScope, scope, element, form;

    beforeEach(angular.mock.module('app.users', 'templates'));

    beforeEach(inject(function(_$compile_, _$rootScope_) {
        $compile = _$compile_;
        $rootScope = _$rootScope_;
        scope = $rootScope.$new();
    }));

    function compileForm() {
        scope.user = {};
        scope.onSave = jasmine.createSpy('onSave');
        element = $compile(
            '<user-form user="user" on-save="onSave($event)"></user-form>'
        )(scope);
        scope.$digest();
        form = scope.$$childHead.userForm;
    }

    it('should be invalid when email is empty', function() {
        compileForm();
        expect(form.email.$error.required).toBe(true);
        expect(form.$valid).toBe(false);
    });

    it('should validate email format', function() {
        compileForm();
        form.email.$setViewValue('not-an-email');
        scope.$digest();
        expect(form.email.$error.email).toBe(true);
    });

    it('should be valid with correct data', function() {
        compileForm();
        form.email.$setViewValue('alice@example.com');
        form.name.$setViewValue('Alice Smith');
        scope.$digest();
        expect(form.$valid).toBe(true);
    });
});
```

---

## Testing Async Operations

### Testing $timeout

```javascript
describe('Debounced search', function() {
    var $timeout, ctrl;

    beforeEach(inject(function(_$timeout_, $componentController) {
        $timeout = _$timeout_;
        ctrl = $componentController('searchBox', null, {
            onSearch: jasmine.createSpy('onSearch')
        });
        ctrl.$onInit();
    }));

    it('should debounce search calls', function() {
        ctrl.onInput('hel');
        ctrl.onInput('hell');
        ctrl.onInput('hello');

        // Only the last call should fire after flush
        $timeout.flush(300);

        expect(ctrl.onSearch).toHaveBeenCalledTimes(1);
        expect(ctrl.onSearch).toHaveBeenCalledWith({ $event: { term: 'hello' } });
    });
});
```

### Testing $interval

```javascript
describe('Polling service', function() {
    var $interval, ctrl;

    beforeEach(inject(function(_$interval_, $componentController) {
        $interval = _$interval_;
        ctrl = $componentController('dashboard', null, {});
        spyOn(ctrl, 'refreshData');
        ctrl.$onInit();
    }));

    it('should poll every 30 seconds', function() {
        $interval.flush(30000);
        expect(ctrl.refreshData).toHaveBeenCalledTimes(1);

        $interval.flush(30000);
        expect(ctrl.refreshData).toHaveBeenCalledTimes(2);
    });

    it('should stop polling on destroy', function() {
        ctrl.$onDestroy();
        $interval.flush(60000);
        expect(ctrl.refreshData).not.toHaveBeenCalled();
    });
});
```

---

## Testing Promises

```javascript
describe('OrderService', function() {
    var OrderService, $rootScope, $q;

    beforeEach(inject(function(_OrderService_, _$rootScope_, _$q_) {
        OrderService = _OrderService_;
        $rootScope = _$rootScope_;
        $q = _$q_;
    }));

    it('should chain promises correctly', function() {
        var result;

        OrderService.createOrder({ items: [{ id: 1, quantity: 2 }] })
            .then(function(order) {
                result = order;
            });

        // Flush $httpBackend if using real HTTP, or resolve promises
        $rootScope.$digest();

        expect(result).toBeDefined();
        expect(result.id).toBeDefined();
    });
});
```

Always call `$rootScope.$digest()` or `$httpBackend.flush()` after setting up promise expectations. AngularJS promises do not resolve until a digest cycle runs.

---

## Cross-Component Integration Testing

```javascript
describe('UserList + UserDetail integration', function() {
    var $compile, $rootScope, $httpBackend, scope;

    beforeEach(angular.mock.module('app.users', 'templates'));

    beforeEach(inject(function(_$compile_, _$rootScope_, _$httpBackend_) {
        $compile = _$compile_;
        $rootScope = _$rootScope_;
        $httpBackend = _$httpBackend_;
        scope = $rootScope.$new();
    }));

    it('should show user detail when a user is selected from the list', function() {
        var users = [
            { id: 1, name: 'Alice' },
            { id: 2, name: 'Bob' }
        ];
        scope.users = users;
        scope.selectedUser = null;

        var element = $compile(
            '<div>' +
            '  <user-list users="users" on-select="selectedUser = $event.user"></user-list>' +
            '  <user-detail ng-if="selectedUser" user="selectedUser"></user-detail>' +
            '</div>'
        )(scope);
        scope.$digest();

        // Click first user
        element.find('.user-card').eq(0).triggerHandler('click');
        scope.$digest();

        expect(scope.selectedUser).toBe(users[0]);
        expect(element.find('user-detail').length).toBe(1);
    });
});
```
