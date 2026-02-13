# SOLID Principles in AngularJS

This reference applies the five SOLID principles to AngularJS-specific patterns with production-ready code examples. Each principle is illustrated with concrete AngularJS patterns that demonstrate both the violation and the correct implementation.

---

## Single Responsibility Principle (SRP)

A module, service, or component should have only one reason to change. In AngularJS, this means separating concerns between controllers, services, directives, and filters.

### Thin Controllers, Fat Services

Move business logic out of controllers into services. The controller's only job is to bind data and delegate actions.

**Violation: Business logic in the controller**

```javascript
function OrderController($http, $log) {
    var vm = this;
    vm.orders = [];

    vm.placeOrder = function(items) {
        var total = items.reduce(function(sum, item) {
            return sum + (item.price * item.quantity);
        }, 0);

        if (total > vm.user.creditLimit) {
            vm.error = 'Credit limit exceeded';
            return;
        }

        $http.post('/api/orders', { items: items, total: total })
            .then(function(response) {
                vm.orders.push(response.data);
                $http.post('/api/notifications', {
                    type: 'order_placed',
                    orderId: response.data.id
                });
            })
            .catch(function(err) {
                $log.error('Order failed', err);
                vm.error = 'Failed to place order';
            });
    };
}
```

**Correct: Service layer handles business logic**

```javascript
// services/order.service.js
function OrderService($http, API_URL) {
    return {
        place: function(items) {
            var total = calculateTotal(items);
            return $http.post(API_URL + '/orders', { items: items, total: total })
                .then(function(response) { return response.data; });
        },
        validateCreditLimit: function(items, creditLimit) {
            return calculateTotal(items) <= creditLimit;
        }
    };

    function calculateTotal(items) {
        return items.reduce(function(sum, item) {
            return sum + (item.price * item.quantity);
        }, 0);
    }
}

// services/notification.service.js
function NotificationService($http, API_URL) {
    return {
        send: function(type, data) {
            return $http.post(API_URL + '/notifications', { type: type, data: data });
        }
    };
}

// Controller — thin, delegation only
function OrderController(OrderService, NotificationService) {
    var vm = this;

    vm.placeOrder = function(items) {
        if (!OrderService.validateCreditLimit(items, vm.user.creditLimit)) {
            vm.error = 'Credit limit exceeded';
            return;
        }

        OrderService.place(items)
            .then(function(order) {
                vm.orders.push(order);
                NotificationService.send('order_placed', { orderId: order.id });
            })
            .catch(function() {
                vm.error = 'Failed to place order';
            });
    };
}
```

### One Directive, One Job

Each directive should handle a single DOM concern.

```javascript
// Bad: directive handles focus, validation, and formatting
// Good: separate directives
angular.module('app.shared')
    .directive('autoFocus', autoFocusDirective)
    .directive('formatCurrency', formatCurrencyDirective)
    .directive('validateRange', validateRangeDirective);
```

---

## Open/Closed Principle (OCP)

Software entities should be open for extension but closed for modification. In AngularJS, achieve this through decorators, interceptor chains, and provider configuration.

### Decorator Pattern

Use `$provide.decorator` to extend existing services without modifying their source code.

```javascript
angular.module('app')
    .config(['$provide', function($provide) {
        $provide.decorator('$log', ['$delegate', function($delegate) {
            var originalWarn = $delegate.warn;

            $delegate.warn = function() {
                // Extend: also send to analytics
                if (window.analytics) {
                    window.analytics.track('warning', { message: arguments[0] });
                }
                originalWarn.apply($delegate, arguments);
            };

            return $delegate;
        }]);
    }]);
```

### Interceptor Chain

Add cross-cutting concerns by registering new interceptors without modifying existing ones.

```javascript
// Each interceptor is independent and composable
$httpProvider.interceptors.push('authInterceptor');
$httpProvider.interceptors.push('errorInterceptor');
$httpProvider.interceptors.push('loadingInterceptor');
$httpProvider.interceptors.push('cachingInterceptor');
```

---

## Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types. In AngularJS, this applies to service interfaces and component contracts.

### Service Interface Consistency

All implementations of a data service must return the same shape.

```javascript
// Both services are substitutable — same interface
function RestUserService($http, API_URL) {
    return {
        getAll: function() {
            return $http.get(API_URL + '/users').then(extractData);
        },
        getById: function(id) {
            return $http.get(API_URL + '/users/' + id).then(extractData);
        }
    };
}

function MockUserService($q) {
    var users = [
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' }
    ];
    return {
        getAll: function() {
            return $q.resolve(users);
        },
        getById: function(id) {
            var user = users.find(function(u) { return u.id === id; });
            return user ? $q.resolve(user) : $q.reject('Not found');
        }
    };
}
```

### Component Binding Contracts

Components that accept the same bindings must behave consistently when substituted.

```javascript
// Both components accept the same bindings and can replace each other
.component('simpleUserCard', {
    bindings: { user: '<', onSelect: '&' },
    template: '<div ng-click="$ctrl.onSelect({$event: {user: $ctrl.user}})">' +
              '<span ng-bind="$ctrl.user.name"></span></div>'
})
.component('detailedUserCard', {
    bindings: { user: '<', onSelect: '&' },
    templateUrl: 'shared/components/detailed-user-card.html'
})
```

---

## Interface Segregation Principle (ISP)

Clients should not be forced to depend on interfaces they do not use. In AngularJS, apply this to service design and component bindings.

### Focused Services

Create separate services for distinct operations instead of one monolithic service.

```javascript
// Bad: one giant service
function UserService($http) {
    return {
        getAll: function() { /* ... */ },
        getById: function() { /* ... */ },
        create: function() { /* ... */ },
        update: function() { /* ... */ },
        delete: function() { /* ... */ },
        validateEmail: function() { /* ... */ },
        sendResetEmail: function() { /* ... */ },
        changePassword: function() { /* ... */ },
        uploadAvatar: function() { /* ... */ },
        getActivityLog: function() { /* ... */ }
    };
}

// Good: segregated by concern
function UserCrudService($http) { /* CRUD operations */ }
function UserAuthService($http) { /* auth, password reset */ }
function UserProfileService($http) { /* avatar, activity */ }
```

### Minimal Component Bindings

Only expose the bindings a component actually needs.

```javascript
// Bad: component receives entire user object when it only needs the name
.component('greeting', {
    bindings: { user: '<' },
    template: '<h1>Hello, {{$ctrl.user.name}}</h1>'
})

// Good: component declares only what it needs
.component('greeting', {
    bindings: { name: '<' },
    template: '<h1>Hello, {{$ctrl.name}}</h1>'
})
```

---

## Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions. In AngularJS, achieve this through service injection and provider-based configuration.

### Provider-Based Injection

Use providers to make the concrete implementation configurable.

```javascript
function StorageProvider() {
    var storageType = 'local';

    this.setStorageType = function(type) {
        storageType = type;
    };

    this.$get = ['$window', function($window) {
        var backends = {
            local: $window.localStorage,
            session: $window.sessionStorage,
            memory: createMemoryStorage()
        };

        return {
            get: function(key) {
                return JSON.parse(backends[storageType].getItem(key));
            },
            set: function(key, value) {
                backends[storageType].setItem(key, JSON.stringify(value));
            },
            remove: function(key) {
                backends[storageType].removeItem(key);
            }
        };
    }];
}

function createMemoryStorage() {
    var store = {};
    return {
        getItem: function(k) { return store[k] || null; },
        setItem: function(k, v) { store[k] = v; },
        removeItem: function(k) { delete store[k]; }
    };
}

// Configuration
angular.module('app')
    .config(['StorageProvider', function(StorageProvider) {
        StorageProvider.setStorageType('local');
    }]);

// In tests
angular.module('app')
    .config(['StorageProvider', function(StorageProvider) {
        StorageProvider.setStorageType('memory');
    }]);
```

### Constant-Based Configuration

Use constants to inject configuration values rather than hardcoding them in services.

```javascript
angular.module('app.core')
    .constant('API_URL', 'https://api.example.com/v1')
    .constant('APP_CONFIG', {
        maxRetries: 3,
        timeout: 5000,
        pageSize: 25
    });

// Services depend on abstract constants, not hardcoded values
function ApiService($http, API_URL, APP_CONFIG) {
    return {
        get: function(endpoint, params) {
            return $http.get(API_URL + endpoint, {
                params: params,
                timeout: APP_CONFIG.timeout
            });
        }
    };
}
```

Swapping the API backend in tests requires only overriding the `API_URL` constant, not modifying any service code.
