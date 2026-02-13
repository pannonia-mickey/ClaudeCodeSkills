---
name: AngularJS Mastery
description: This skill should be used when the user asks about "AngularJS module", "AngularJS directive", "AngularJS component", "AngularJS service", "AngularJS factory", "controllerAs", "ui-router", "AngularJS form", or "ng-model". It covers AngularJS 1.5+ core patterns including module organization, component architecture, directive design, service recipes, routing with ui-router, form validation, and HTTP communication. Use this skill for structuring an AngularJS application, choosing between components and directives, designing services, implementing routing, or building validated forms.
---

## Module Organization

Organize AngularJS applications into feature modules with explicit dependency declarations. Never retrieve a module by calling `angular.module('name')` without the dependency array — always define modules once and reference them via dependency injection.

```
app/
    app.module.js
    app.config.js
    app.run.js
    core/
        core.module.js
        services/
            auth.service.js
            api.service.js
            logger.service.js
        interceptors/
            auth.interceptor.js
            error.interceptor.js
        constants/
            api.constants.js
    shared/
        shared.module.js
        directives/
            focus-on.directive.js
            click-outside.directive.js
        filters/
            truncate.filter.js
            time-ago.filter.js
        components/
            loading-spinner/
                loading-spinner.component.js
                loading-spinner.html
                loading-spinner.css
    features/
        dashboard/
            dashboard.module.js
            dashboard.route.js
            dashboard.component.js
            dashboard.html
            widgets/
                stats-card.component.js
                activity-feed.component.js
        users/
            users.module.js
            users.route.js
            user-list.component.js
            user-detail.component.js
            user.service.js
```

```javascript
// app.module.js
angular.module('app', [
    'app.core',
    'app.shared',
    'app.dashboard',
    'app.users'
]);
```

```javascript
// core/core.module.js
angular.module('app.core', [
    'ngSanitize',
    'ngMessages',
    'ui.router'
]);
```

```javascript
// features/dashboard/dashboard.module.js
angular.module('app.dashboard', [
    'app.core',
    'app.shared'
]);
```

Each feature module declares only the dependencies it directly uses. The root module composes all feature modules.

## Component Architecture (1.5+)

Prefer `.component()` for all UI elements. Components enforce one-way data flow, use isolate scope by default, and provide lifecycle hooks.

```javascript
// features/users/user-list.component.js
(function() {
    'use strict';

    angular
        .module('app.users')
        .component('userList', {
            templateUrl: 'features/users/user-list.html',
            controller: UserListController,
            bindings: {
                users: '<',
                onSelect: '&'
            }
        });

    UserListController.$inject = ['UserService', '$log'];

    function UserListController(UserService, $log) {
        var $ctrl = this;

        $ctrl.searchTerm = '';
        $ctrl.filteredUsers = [];

        $ctrl.$onInit = function() {
            $ctrl.filteredUsers = $ctrl.users || [];
            $log.debug('UserListController initialized with', $ctrl.users.length, 'users');
        };

        $ctrl.$onChanges = function(changes) {
            if (changes.users && changes.users.currentValue) {
                $ctrl.filteredUsers = filterUsers($ctrl.users, $ctrl.searchTerm);
            }
        };

        $ctrl.search = function() {
            $ctrl.filteredUsers = filterUsers($ctrl.users, $ctrl.searchTerm);
        };

        $ctrl.selectUser = function(user) {
            $ctrl.onSelect({ $event: { user: user } });
        };

        function filterUsers(users, term) {
            if (!term) return users;
            var lower = term.toLowerCase();
            return users.filter(function(user) {
                return user.name.toLowerCase().indexOf(lower) > -1;
            });
        }
    }
})();
```

```html
<!-- features/users/user-list.html -->
<div class="user-list">
    <input type="text"
           ng-model="$ctrl.searchTerm"
           ng-change="$ctrl.search()"
           ng-model-options="{ debounce: 300 }"
           placeholder="Search users..." />

    <div class="user-card"
         ng-repeat="user in $ctrl.filteredUsers track by user.id"
         ng-click="$ctrl.selectUser(user)">
        <span ng-bind="user.name"></span>
        <span ng-bind="user.email"></span>
    </div>

    <p ng-if="$ctrl.filteredUsers.length === 0">No users found.</p>
</div>
```

### Component Bindings

| Symbol | Type | Direction | Use Case |
|--------|------|-----------|----------|
| `<` | One-way | Parent → Child | Input data (preferred) |
| `@` | String | Parent → Child | Interpolated string attributes |
| `&` | Expression | Child → Parent | Callback events |
| `=` | Two-way | Bidirectional | Avoid — use `<` and `&` instead |

Always prefer `<` (one-way) over `=` (two-way). Use `&` for output events. Two-way binding creates hidden coupling and makes data flow unpredictable.

## Directive Design

Use directives for DOM manipulation, behavioral extensions, and structural templating. Components handle UI; directives handle behavior.

```javascript
// shared/directives/click-outside.directive.js
(function() {
    'use strict';

    angular
        .module('app.shared')
        .directive('clickOutside', clickOutside);

    clickOutside.$inject = ['$document', '$parse'];

    function clickOutside($document, $parse) {
        return {
            restrict: 'A',
            link: function(scope, element, attrs) {
                var handler = $parse(attrs.clickOutside);

                function onClick(event) {
                    if (!element[0].contains(event.target)) {
                        scope.$apply(function() {
                            handler(scope);
                        });
                    }
                }

                $document.on('click', onClick);

                scope.$on('$destroy', function() {
                    $document.off('click', onClick);
                });
            }
        };
    }
})();
```

```javascript
// shared/directives/focus-on.directive.js
(function() {
    'use strict';

    angular
        .module('app.shared')
        .directive('focusOn', focusOn);

    focusOn.$inject = ['$timeout'];

    function focusOn($timeout) {
        return {
            restrict: 'A',
            link: function(scope, element, attrs) {
                scope.$watch(attrs.focusOn, function(value) {
                    if (value) {
                        $timeout(function() {
                            element[0].focus();
                        });
                    }
                });
            }
        };
    }
})();
```

### Directive Restrict Values

| Value | Element/Attribute | When to Use |
|-------|------------------|-------------|
| `'E'` | Element `<my-dir>` | Standalone components (prefer `.component()`) |
| `'A'` | Attribute `<div my-dir>` | Behavioral/decorating directives |
| `'C'` | Class `<div class="my-dir">` | Avoid — poor readability |
| `'EA'` | Both | Avoid — pick one |

## Service Recipes

### Service vs Factory vs Provider

```javascript
// Service — constructor function, instantiated with `new`
angular.module('app.core')
    .service('AuthService', AuthService);

AuthService.$inject = ['$http', 'API_URL'];

function AuthService($http, API_URL) {
    this.login = function(credentials) {
        return $http.post(API_URL + '/auth/login', credentials);
    };
    this.logout = function() {
        return $http.post(API_URL + '/auth/logout');
    };
    this.getCurrentUser = function() {
        return $http.get(API_URL + '/auth/me');
    };
}
```

```javascript
// Factory — returns an object, more flexible
angular.module('app.core')
    .factory('UserService', UserService);

UserService.$inject = ['$http', 'API_URL', '$log'];

function UserService($http, API_URL, $log) {
    var cache = {};

    return {
        getAll: getAll,
        getById: getById,
        create: create,
        update: update,
        clearCache: clearCache
    };

    function getAll(params) {
        return $http.get(API_URL + '/users', { params: params })
            .then(function(response) { return response.data; });
    }

    function getById(id) {
        if (cache[id]) {
            return $q.resolve(cache[id]);
        }
        return $http.get(API_URL + '/users/' + id)
            .then(function(response) {
                cache[id] = response.data;
                return response.data;
            });
    }

    function create(userData) {
        return $http.post(API_URL + '/users', userData)
            .then(function(response) { return response.data; });
    }

    function update(id, userData) {
        return $http.put(API_URL + '/users/' + id, userData)
            .then(function(response) {
                cache[id] = response.data;
                return response.data;
            });
    }

    function clearCache() {
        cache = {};
    }
}
```

```javascript
// Provider — configurable in .config() phase
angular.module('app.core')
    .provider('Logger', LoggerProvider);

function LoggerProvider() {
    var logLevel = 'info';
    var levels = { debug: 0, info: 1, warn: 2, error: 3 };

    this.setLogLevel = function(level) {
        if (levels.hasOwnProperty(level)) {
            logLevel = level;
        }
    };

    this.$get = ['$log', function($log) {
        return {
            debug: function(msg) { if (levels[logLevel] <= 0) $log.debug(msg); },
            info: function(msg) { if (levels[logLevel] <= 1) $log.info(msg); },
            warn: function(msg) { if (levels[logLevel] <= 2) $log.warn(msg); },
            error: function(msg) { if (levels[logLevel] <= 3) $log.error(msg); }
        };
    }];
}

// Configuration
angular.module('app')
    .config(['LoggerProvider', function(LoggerProvider) {
        LoggerProvider.setLogLevel('warn');
    }]);
```

| Recipe | Config Phase | Instance | Use Case |
|--------|-------------|----------|----------|
| `constant` | Available | Singleton | Static values, config-time data |
| `value` | Not available | Singleton | Runtime-only constants |
| `factory` | Not available | Singleton | Object creation with private state |
| `service` | Not available | Singleton | Constructor-based singleton |
| `provider` | Available (provider instance) | Singleton | Configurable services |

## Routing with ui-router

```javascript
// features/users/users.route.js
(function() {
    'use strict';

    angular
        .module('app.users')
        .config(routeConfig);

    routeConfig.$inject = ['$stateProvider'];

    function routeConfig($stateProvider) {
        $stateProvider
            .state('users', {
                url: '/users',
                component: 'usersView',
                resolve: {
                    users: ['UserService', function(UserService) {
                        return UserService.getAll();
                    }]
                }
            })
            .state('users.detail', {
                url: '/:userId',
                component: 'userDetail',
                resolve: {
                    user: ['UserService', '$transition$', function(UserService, $transition$) {
                        return UserService.getById($transition$.params().userId);
                    }]
                }
            })
            .state('users.create', {
                url: '/new',
                component: 'userForm',
                resolve: {
                    user: function() { return {}; }
                }
            });
    }
})();
```

### Transition Hooks for Guards

```javascript
// core/auth-guard.run.js
angular.module('app.core')
    .run(authGuard);

authGuard.$inject = ['$transitions', 'AuthService'];

function authGuard($transitions, AuthService) {
    $transitions.onBefore({ to: function(state) {
        return state.data && state.data.requiresAuth;
    }}, function(transition) {
        var AuthService = transition.injector().get('AuthService');
        if (!AuthService.isAuthenticated()) {
            return transition.router.stateService.target('login', {
                returnTo: transition.to().name
            });
        }
    });
}
```

## Form Validation

```javascript
// features/users/user-form.component.js
angular.module('app.users')
    .component('userForm', {
        templateUrl: 'features/users/user-form.html',
        controller: UserFormController,
        bindings: {
            user: '<',
            onSave: '&'
        }
    });

UserFormController.$inject = ['UserService'];

function UserFormController(UserService) {
    var $ctrl = this;

    $ctrl.$onInit = function() {
        $ctrl.formData = angular.copy($ctrl.user) || {};
    };

    $ctrl.submit = function(form) {
        if (form.$invalid) return;

        $ctrl.saving = true;
        $ctrl.onSave({ $event: { user: $ctrl.formData } });
    };
}
```

```html
<!-- features/users/user-form.html -->
<form name="userForm" ng-submit="$ctrl.submit(userForm)" novalidate>
    <div class="form-group">
        <label for="email">Email</label>
        <input type="email" id="email" name="email"
               ng-model="$ctrl.formData.email"
               required
               unique-email />
        <div ng-messages="userForm.email.$error" ng-if="userForm.email.$touched">
            <p ng-message="required">Email is required.</p>
            <p ng-message="email">Enter a valid email.</p>
            <p ng-message="uniqueEmail">This email is already taken.</p>
        </div>
    </div>

    <div class="form-group">
        <label for="name">Name</label>
        <input type="text" id="name" name="name"
               ng-model="$ctrl.formData.name"
               required
               ng-minlength="2"
               ng-maxlength="100" />
        <div ng-messages="userForm.name.$error" ng-if="userForm.name.$touched">
            <p ng-message="required">Name is required.</p>
            <p ng-message="minlength">Name must be at least 2 characters.</p>
            <p ng-message="maxlength">Name cannot exceed 100 characters.</p>
        </div>
    </div>

    <button type="submit" ng-disabled="userForm.$invalid || $ctrl.saving">
        {{ $ctrl.saving ? 'Saving...' : 'Save' }}
    </button>
</form>
```

### Custom Async Validator

```javascript
// shared/directives/unique-email.directive.js
angular.module('app.shared')
    .directive('uniqueEmail', uniqueEmail);

uniqueEmail.$inject = ['UserService', '$q'];

function uniqueEmail(UserService, $q) {
    return {
        restrict: 'A',
        require: 'ngModel',
        link: function(scope, element, attrs, ngModel) {
            ngModel.$asyncValidators.uniqueEmail = function(modelValue) {
                if (!modelValue) return $q.resolve();

                return UserService.checkEmail(modelValue)
                    .then(function(response) {
                        if (response.data.exists) {
                            return $q.reject('taken');
                        }
                        return true;
                    });
            };
        }
    };
}
```

## HTTP Interceptors

```javascript
// core/interceptors/auth.interceptor.js
angular.module('app.core')
    .factory('authInterceptor', authInterceptor);

authInterceptor.$inject = ['$injector', '$q'];

function authInterceptor($injector, $q) {
    return {
        request: function(config) {
            var TokenService = $injector.get('TokenService');
            var token = TokenService.getToken();
            if (token) {
                config.headers.Authorization = 'Bearer ' + token;
            }
            return config;
        },
        responseError: function(rejection) {
            if (rejection.status === 401) {
                var $state = $injector.get('$state');
                $state.go('login');
            }
            return $q.reject(rejection);
        }
    };
}

// Register interceptor
angular.module('app.core')
    .config(['$httpProvider', function($httpProvider) {
        $httpProvider.interceptors.push('authInterceptor');
    }]);
```

---

## References

For AngularJS-specific SOLID design principles and architectural patterns, see [solid-patterns.md](references/solid-patterns.md).

For AngularJS best practices covering project conventions, coding standards, and common pitfalls, see [best-practices.md](references/best-practices.md).
