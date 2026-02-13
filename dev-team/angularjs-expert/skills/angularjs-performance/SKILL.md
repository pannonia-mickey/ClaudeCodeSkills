---
name: AngularJS Performance
description: This skill should be used when the user asks about "AngularJS performance", "digest cycle", "watcher count", "ng-repeat slow", "one-time binding", "AngularJS lazy loading", "$applyAsync", or "AngularJS memory leak". It covers AngularJS performance optimization including digest cycle tuning, watcher reduction strategies, one-time bindings, ng-repeat optimization, lazy loading with ocLazyLoad, memory leak prevention, $http caching, and rendering performance with virtual scrolling.
---

## Digest Cycle Fundamentals

The digest cycle is AngularJS's change detection mechanism. On each cycle, AngularJS iterates through all registered watchers and checks if their values have changed. Performance degrades linearly with the number of watchers.

### Performance Thresholds

| Watcher Count | Performance | Action |
|--------------|-------------|--------|
| < 200 | Excellent | No optimization needed |
| 200–500 | Good | Monitor, optimize hot paths |
| 500–1000 | Degraded | Active optimization required |
| 1000–2000 | Poor | Major refactoring needed |
| > 2000 | Unusable | Architectural rethink |

### Counting Watchers

```javascript
// Debug utility — count all watchers on the page
function countWatchers() {
    var root = angular.element(document.getElementsByTagName('body'));
    var count = 0;

    function visit(element) {
        var scope = element.data('$scope');
        var isolateScope = element.data('$isolateScope');

        if (scope) count += (scope.$$watchers || []).length;
        if (isolateScope) count += (isolateScope.$$watchers || []).length;

        angular.forEach(element.children(), function(child) {
            visit(angular.element(child));
        });
    }

    visit(root);
    return count;
}

// Use in browser console or AngularJS Batarang extension
```

---

## Watcher Reduction Strategies

### One-Time Bindings (:: syntax)

Use the `::` prefix for values that do not change after initial rendering. The watcher is automatically deregistered once the value stabilizes.

```html
<!-- Two-way binding: watcher persists forever -->
<span>{{ $ctrl.user.name }}</span>

<!-- One-time binding: watcher removed after first stable value -->
<span>{{ ::$ctrl.user.name }}</span>
<span ng-bind="::$ctrl.user.name"></span>

<!-- One-time binding in ng-repeat -->
<div ng-repeat="item in ::$ctrl.items track by item.id">
    <span ng-bind="::item.name"></span>
    <span ng-bind="::item.price | currency"></span>
</div>

<!-- One-time binding in ng-if/ng-show -->
<div ng-if="::$ctrl.isAdmin">Admin Panel</div>
```

Use one-time bindings for:
- Static labels and headings
- Data displayed in read-only lists
- Configuration values that do not change
- User profile information on non-editable pages

Do NOT use one-time bindings for:
- Form inputs bound with `ng-model`
- Values that update in real-time (counters, timers)
- Conditional displays that toggle based on user actions

### Deregistering Watchers Manually

```javascript
$ctrl.$onInit = function() {
    // $watch returns a deregistration function
    var unwatch = $scope.$watch('$ctrl.searchTerm', function(newVal) {
        if (newVal && newVal.length >= 3) {
            $ctrl.performSearch(newVal);
            // Optionally deregister after first match
            // unwatch();
        }
    });

    // Always clean up in $onDestroy
    $ctrl.$onDestroy = function() {
        unwatch();
    };
};
```

### Replacing $watch with $onChanges

Components with input bindings should use `$onChanges` instead of `$scope.$watch`. The framework handles change detection more efficiently.

```javascript
// Bad: manual watcher inside component
$scope.$watch('$ctrl.userId', function(newId) {
    if (newId) loadUser(newId);
});

// Good: lifecycle hook — no extra watcher
$ctrl.$onChanges = function(changes) {
    if (changes.userId && changes.userId.currentValue) {
        loadUser(changes.userId.currentValue);
    }
};
```

---

## ng-repeat Optimization

### track by

Always use `track by` to tell AngularJS how to identify unique items. Without it, AngularJS destroys and recreates all DOM elements on every data refresh.

```html
<!-- Bad: entire DOM recreated on refresh -->
<div ng-repeat="product in $ctrl.products">

<!-- Good: reuses DOM elements for unchanged items -->
<div ng-repeat="product in $ctrl.products track by product.id">

<!-- For server-generated data with no unique ID -->
<div ng-repeat="item in $ctrl.items track by $index">
```

### Limit Displayed Items

For large datasets, limit the visible items and implement pagination or infinite scroll.

```html
<!-- Client-side pagination -->
<div ng-repeat="item in $ctrl.items | limitTo: $ctrl.pageSize : ($ctrl.currentPage * $ctrl.pageSize) track by item.id">
```

```javascript
// Virtual scrolling with vs-repeat
// <div vs-repeat>
//     <div ng-repeat="item in $ctrl.items track by item.id">
//         {{ ::item.name }}
//     </div>
// </div>
```

### Avoid Filters in ng-repeat

Filters inside `ng-repeat` execute on every digest cycle. Move filtering to the controller.

```html
<!-- Bad: filter runs on every digest -->
<div ng-repeat="user in $ctrl.users | filter: $ctrl.searchTerm | orderBy: 'name'">

<!-- Good: pre-filtered in controller -->
<div ng-repeat="user in $ctrl.filteredUsers track by user.id">
```

```javascript
$ctrl.search = function() {
    $ctrl.filteredUsers = $ctrl.users.filter(function(user) {
        return user.name.toLowerCase().indexOf($ctrl.searchTerm.toLowerCase()) > -1;
    });
};
```

---

## $http Performance

### $applyAsync for Batch Digest

When the application makes many concurrent HTTP requests, each response triggers a separate digest cycle. Use `$applyAsync` to batch them.

```javascript
angular.module('app')
    .config(['$httpProvider', function($httpProvider) {
        // Batches multiple $http responses into a single digest cycle
        $httpProvider.useApplyAsync(true);
    }]);
```

This adds a ~10ms delay before triggering the digest but dramatically reduces digest cycle count when loading dashboards with many API calls.

### Response Caching

```javascript
// Built-in $http cache
function DataService($http, $cacheFactory, API_URL) {
    var dataCache = $cacheFactory('dataCache');

    return {
        getCategories: function() {
            return $http.get(API_URL + '/categories', { cache: dataCache })
                .then(function(response) { return response.data; });
        },
        invalidateCache: function() {
            dataCache.removeAll();
        }
    };
}
```

### Request Deduplication

Prevent duplicate concurrent requests for the same resource.

```javascript
function DeduplicatedService($http, $q, API_URL) {
    var pendingRequests = {};

    return {
        get: function(endpoint) {
            if (pendingRequests[endpoint]) {
                return pendingRequests[endpoint];
            }

            var promise = $http.get(API_URL + endpoint)
                .then(function(response) { return response.data; })
                .finally(function() {
                    delete pendingRequests[endpoint];
                });

            pendingRequests[endpoint] = promise;
            return promise;
        }
    };
}
```

---

## ng-model-options for Input Performance

Debounce model updates to reduce digest cycles during fast typing.

```html
<!-- Triggers digest on every keystroke (bad for search fields) -->
<input ng-model="$ctrl.searchTerm" />

<!-- Debounce: triggers digest 300ms after user stops typing -->
<input ng-model="$ctrl.searchTerm"
       ng-model-options="{ debounce: 300 }"
       ng-change="$ctrl.search()" />

<!-- Different debounce for different events -->
<input ng-model="$ctrl.searchTerm"
       ng-model-options="{ debounce: { default: 300, blur: 0 } }"
       ng-change="$ctrl.search()" />
```

---

## Memory Leak Prevention

### Common Leak Sources

| Source | Cause | Prevention |
|--------|-------|------------|
| `$scope.$on` | Listener not removed | Store and call deregistration function in `$onDestroy` |
| `$scope.$watch` | Watcher not deregistered | Store and call deregistration function in `$onDestroy` |
| `$interval` | Interval not cancelled | Call `$interval.cancel()` in `$onDestroy` |
| `$timeout` | Timeout not cancelled | Call `$timeout.cancel()` in `$onDestroy` |
| DOM event listeners | `element.on()` not cleaned | Call `element.off()` in `$onDestroy` or link `$destroy` |
| Closures | Service holds reference to destroyed scope | Use weak references or nullify in `$onDestroy` |

### Cleanup Pattern

```javascript
function DashboardController($scope, $interval, $timeout, EventBus) {
    var $ctrl = this;
    var pollingHandle;
    var debounceHandle;
    var eventUnsubscribe;

    $ctrl.$onInit = function() {
        pollingHandle = $interval(refreshData, 30000);

        eventUnsubscribe = $scope.$on('data:updated', function(event, data) {
            $ctrl.data = data;
        });
    };

    $ctrl.onSearch = function(term) {
        if (debounceHandle) $timeout.cancel(debounceHandle);
        debounceHandle = $timeout(function() {
            performSearch(term);
        }, 300);
    };

    $ctrl.$onDestroy = function() {
        $interval.cancel(pollingHandle);
        if (debounceHandle) $timeout.cancel(debounceHandle);
        if (eventUnsubscribe) eventUnsubscribe();
    };
}
```

---

## Lazy Loading with ocLazyLoad

Load feature modules on demand to reduce initial bundle size.

```javascript
// Route-based lazy loading with ui-router and ocLazyLoad
$stateProvider.state('reports', {
    url: '/reports',
    templateUrl: 'features/reports/reports.html',
    controller: 'ReportsController',
    controllerAs: 'vm',
    resolve: {
        loadModule: ['$ocLazyLoad', function($ocLazyLoad) {
            return $ocLazyLoad.load({
                name: 'app.reports',
                files: [
                    'features/reports/reports.module.js',
                    'features/reports/reports.controller.js',
                    'features/reports/reports.service.js'
                ]
            });
        }]
    }
});
```

---

## References

- **[Digest Cycle Deep Dive](references/digest-cycle-internals.md)** — Internal mechanics of `$digest`, `$apply`, `$evalAsync`, and `$applyAsync`, including profiling techniques and optimization strategies.
- **[Rendering Optimization](references/rendering-optimization.md)** — Virtual scrolling, DOM recycling, `ng-repeat` alternatives, and rendering pipeline tuning for large datasets.
