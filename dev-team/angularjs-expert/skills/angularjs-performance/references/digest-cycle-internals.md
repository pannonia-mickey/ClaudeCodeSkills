# Digest Cycle Internals

Comprehensive reference covering the internal mechanics of AngularJS's dirty-checking system, profiling techniques, and advanced optimization strategies.

---

## How the Digest Cycle Works

### Execution Flow

1. An event occurs (user click, HTTP response, timer, etc.)
2. AngularJS calls `$apply()` which calls `$rootScope.$digest()`
3. `$digest()` iterates through all `$$watchers` on the current scope and its children
4. For each watcher, it compares `oldValue` with `newValue` (dirty checking)
5. If any watcher's value changed, the digest loop runs again (dirty → re-check)
6. The loop repeats until no values change or the TTL limit (10 iterations) is reached
7. If the TTL is exceeded, AngularJS throws an "infdig" (infinite digest) error

### $apply vs $digest vs $evalAsync vs $applyAsync

| Method | Scope | Triggers Digest | Use Case |
|--------|-------|----------------|----------|
| `$apply(fn)` | From `$rootScope` | Full digest from root | External events (DOM, third-party libs) |
| `$digest()` | Current scope + children | Partial digest | Internal — rarely call directly |
| `$evalAsync(fn)` | Current scope | Runs in current/next digest | Safe alternative to `$apply` inside digest |
| `$applyAsync(fn)` | From `$rootScope` | Batched, delayed | Multiple rapid external events |

### When to Use Each

```javascript
// External event handler (non-AngularJS context)
element.addEventListener('resize', function() {
    scope.$apply(function() {
        scope.width = element.offsetWidth;
    });
});

// Inside a directive link function that may be in a digest
scope.$evalAsync(function() {
    scope.ready = true;
});

// Batching multiple updates from WebSocket messages
socket.on('update', function(data) {
    scope.$applyAsync(function() {
        scope.data = data;
    });
});
```

---

## Profiling the Digest Cycle

### Console Timing

```javascript
// Monkey-patch $digest to measure duration
angular.module('app').run(['$rootScope', function($rootScope) {
    var original = $rootScope.$digest;
    $rootScope.$digest = function() {
        var start = performance.now();
        original.apply(this, arguments);
        var duration = performance.now() - start;
        if (duration > 50) {
            console.warn('Slow digest cycle:', duration.toFixed(2) + 'ms');
        }
    };
}]);
```

### Watcher Cost Profiling

```javascript
// Identify the most expensive watchers
function profileWatchers() {
    var root = angular.element(document.body).scope();
    var watchers = [];

    function collectWatchers(scope) {
        if (scope.$$watchers) {
            scope.$$watchers.forEach(function(w) {
                var start = performance.now();
                try { w.get(scope); } catch (e) {}
                var cost = performance.now() - start;
                watchers.push({ exp: w.exp || w.fn, cost: cost, scope: scope.$id });
            });
        }
        if (scope.$$childHead) collectWatchers(scope.$$childHead);
        if (scope.$$nextSibling) collectWatchers(scope.$$nextSibling);
    }

    collectWatchers(root);
    return watchers.sort(function(a, b) { return b.cost - a.cost; }).slice(0, 20);
}
```

### Chrome DevTools Performance Tab

1. Open Chrome DevTools → Performance tab
2. Click Record
3. Trigger the slow interaction
4. Stop recording
5. Search for `$digest` in the flame chart
6. Look for long `$digest` blocks and identify the callers

---

## Advanced Optimization Strategies

### Scope Isolation to Reduce Watcher Propagation

Child scopes inherit from parent scopes. When `$digest` runs, it walks the entire scope tree. Components with isolate scope limit the watcher search space.

```javascript
// Using .component() creates isolate scope automatically
// This limits digest traversal to this component's subtree
angular.module('app.shared')
    .component('stockTicker', {
        bindings: { symbol: '<' },
        controller: StockTickerController,
        template: '<span ng-bind="$ctrl.price"></span>'
    });
```

### $watchCollection vs Deep $watch

```javascript
// $watch with deep comparison — most expensive
$scope.$watch('$ctrl.items', callback, true);  // O(n) deep comparison every cycle

// $watchCollection — moderate cost, catches add/remove/reorder
$scope.$watchCollection('$ctrl.items', callback);  // Shallow comparison of array

// Specific property watch — cheapest
$scope.$watch('$ctrl.items.length', callback);  // Single value comparison
```

| Watch Type | Cost | Detects |
|-----------|------|---------|
| `$watch(expr)` | Low | Reference change |
| `$watch(expr, fn, true)` | High | Deep value change |
| `$watchCollection(expr)` | Medium | Add, remove, reorder |
| `$watchGroup([...])` | Low | Multiple reference changes |

### Suspend and Resume Digest

For off-screen components, suspend the scope to skip it during digest cycles.

```javascript
// AngularJS 1.8+
function TabPanelController($scope) {
    var $ctrl = this;

    $ctrl.onTabDeactivate = function() {
        $scope.$suspend();  // Skip this scope in digest cycles
    };

    $ctrl.onTabActivate = function() {
        $scope.$resume();   // Re-include in digest cycles
        $scope.$digest();   // Catch up on missed changes
    };
}
```

---

## Infinite Digest Prevention

### Common Causes

1. **Returning new objects from getters**: Each digest sees a "new" value
2. **Filters that create new arrays**: `filter` and `orderBy` return new arrays
3. **Watch expressions with side effects**: Watcher callback modifies watched value

### Fix: Stabilize Return Values

```javascript
// Bad: returns new object every time → infinite digest
$ctrl.getUser = function() {
    return { name: $ctrl.firstName + ' ' + $ctrl.lastName };
};
// <span>{{ $ctrl.getUser().name }}</span>

// Good: compute and store the value
$ctrl.$onChanges = function() {
    $ctrl.user = { name: $ctrl.firstName + ' ' + $ctrl.lastName };
};
// <span ng-bind="$ctrl.user.name"></span>
```

### Fix: Cache Filter Results

```javascript
// Bad: orderBy creates new array on every digest
// ng-repeat="item in $ctrl.items | orderBy:'name'"

// Good: sort in controller, bind the result
$ctrl.sortedItems = angular.copy($ctrl.items).sort(function(a, b) {
    return a.name.localeCompare(b.name);
});
// ng-repeat="item in $ctrl.sortedItems track by item.id"
```
