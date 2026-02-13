# AngularJS Best Practices

Comprehensive reference covering coding conventions, common pitfalls, dependency injection safety, and project conventions for AngularJS 1.5+ applications.

---

## Coding Conventions

### IIFE Wrapping

Wrap every file in an Immediately Invoked Function Expression to avoid polluting the global scope.

```javascript
(function() {
    'use strict';

    angular
        .module('app.users')
        .component('userProfile', {
            templateUrl: 'features/users/user-profile.html',
            controller: UserProfileController,
            bindings: { userId: '<' }
        });

    UserProfileController.$inject = ['UserService'];

    function UserProfileController(UserService) {
        var $ctrl = this;
        // ...
    }
})();
```

### Named Functions Over Anonymous

Use named function declarations for controllers, services, and directive definitions. This improves stack traces and debugging.

```javascript
// Good: named function
angular.module('app.core')
    .factory('AuthService', AuthService);

function AuthService($http) { /* ... */ }

// Bad: anonymous function
angular.module('app.core')
    .factory('AuthService', function($http) { /* ... */ });
```

### Minification-Safe Dependency Injection

Always use `$inject` property annotation. Never rely on implicit annotation.

```javascript
// Good: explicit $inject
UserController.$inject = ['UserService', '$state', '$log'];

function UserController(UserService, $state, $log) {
    // ...
}

// Also acceptable: /* @ngInject */ annotation (with ng-annotate build tool)
/* @ngInject */
function UserController(UserService, $state, $log) {
    // ...
}

// Bad: implicit annotation (breaks on minification)
function UserController(UserService, $state, $log) {
    // ...
}
```

### controllerAs Always

Never inject `$scope` into controllers. Use `controllerAs` syntax exclusively.

```javascript
// Good
.component('userList', {
    controller: UserListController,
    controllerAs: '$ctrl'  // default in .component()
})

// Good: for routes using controller directly
$stateProvider.state('users', {
    controller: 'UserListController',
    controllerAs: 'vm'
});

// Bad: $scope injection
function UserListController($scope) {
    $scope.users = [];  // avoid this
}
```

---

## Common Pitfalls

### 1. Forgotten Cleanup

Always clean up `$on` listeners, `$watch` registrations, `$interval`, and `$timeout` in `$onDestroy`.

```javascript
function DashboardController($interval, $scope, StatsService) {
    var $ctrl = this;
    var pollInterval;
    var unsubscribe;

    $ctrl.$onInit = function() {
        pollInterval = $interval(function() {
            StatsService.fetch().then(function(data) {
                $ctrl.stats = data;
            });
        }, 30000);

        unsubscribe = $scope.$on('refresh-dashboard', function() {
            StatsService.fetch().then(function(data) {
                $ctrl.stats = data;
            });
        });
    };

    $ctrl.$onDestroy = function() {
        if (pollInterval) $interval.cancel(pollInterval);
        if (unsubscribe) unsubscribe();
    };
}
```

### 2. $apply Already In Progress

Never call `$apply` or `$digest` inside code that may already be in a digest cycle. Use `$applyAsync` or `$evalAsync` instead.

```javascript
// Bad: may cause "$apply already in progress"
element.on('click', function() {
    scope.$apply(function() {
        scope.clicked = true;
    });
});

// Good: safe in all contexts
element.on('click', function() {
    scope.$evalAsync(function() {
        scope.clicked = true;
    });
});
```

### 3. Functions in Templates

Never bind functions directly in template expressions. They execute on every digest cycle.

```javascript
// Bad: getFullName() runs on every digest
// <span>{{ $ctrl.getFullName() }}</span>

// Good: compute once and bind the result
$ctrl.$onInit = function() {
    $ctrl.fullName = $ctrl.firstName + ' ' + $ctrl.lastName;
};
// <span ng-bind="$ctrl.fullName"></span>

// Good: if reactive, use $onChanges
$ctrl.$onChanges = function(changes) {
    if (changes.firstName || changes.lastName) {
        $ctrl.fullName = $ctrl.firstName + ' ' + $ctrl.lastName;
    }
};
```

### 4. ng-repeat Without track by

Always use `track by` to prevent unnecessary DOM destruction and recreation.

```javascript
// Bad: destroys and recreates all DOM elements on data refresh
// ng-repeat="item in $ctrl.items"

// Good: reuses DOM elements for unchanged items
// ng-repeat="item in $ctrl.items track by item.id"

// For primitive arrays
// ng-repeat="tag in $ctrl.tags track by $index"
```

### 5. Circular Dependencies

Avoid circular service injection. Use `$injector` for lazy resolution when needed.

```javascript
// Bad: ServiceA depends on ServiceB and vice versa
// This causes an AngularJS circular dependency error

// Good: use $injector for lazy resolution
function ServiceA($injector) {
    return {
        doSomething: function() {
            var ServiceB = $injector.get('ServiceB');
            return ServiceB.process();
        }
    };
}
```

---

## File Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Module | `feature.module.js` | `users.module.js` |
| Component | `feature-name.component.js` | `user-list.component.js` |
| Directive | `behavior-name.directive.js` | `click-outside.directive.js` |
| Service | `name.service.js` | `auth.service.js` |
| Factory | `name.factory.js` or `name.service.js` | `user.service.js` |
| Filter | `name.filter.js` | `truncate.filter.js` |
| Route | `feature.route.js` | `users.route.js` |
| Template | `component-name.html` | `user-list.html` |
| Test | `feature-name.spec.js` | `user-list.spec.js` |

---

## Build and Tooling

### Recommended Build Stack

- **Task runner**: Gulp or Webpack with `angular-webpack-plugin`
- **Template cache**: `gulp-angular-templatecache` or `ngtemplate-loader` to pre-load templates
- **Annotation**: `ng-annotate` (Babel plugin or Gulp plugin) for automatic `$inject` annotation
- **Linting**: ESLint with `eslint-plugin-angular`
- **CSS**: Component-scoped styles via BEM naming or CSS Modules

### Template Caching

Pre-compile templates into the `$templateCache` to avoid HTTP requests:

```javascript
// Generated by gulp-angular-templatecache
angular.module('app').run(['$templateCache', function($templateCache) {
    $templateCache.put('features/users/user-list.html',
        '<div class="user-list">...</div>');
}]);
```

---

## One File Per Concept

Every file should contain exactly one AngularJS concept: one component, one service, one directive, or one filter. Never define multiple components or services in a single file.

```
// Good
user-list.component.js    → defines userList component
user.service.js           → defines UserService
truncate.filter.js        → defines truncate filter

// Bad
users.js                  → defines userList component AND UserService AND truncate filter
```
