# Rendering Optimization

Comprehensive reference covering virtual scrolling, DOM recycling, ng-repeat alternatives, and rendering pipeline tuning for AngularJS applications with large datasets.

---

## Virtual Scrolling

### Problem

Rendering thousands of DOM elements in `ng-repeat` causes:
- High initial render time (each element creates watchers)
- Memory consumption (each element holds a scope)
- Slow digest cycles (every watcher evaluated on every cycle)

### Solution: vs-repeat (angular-vs-repeat)

Only render the visible portion of the list. DOM elements are recycled as the user scrolls.

```html
<div vs-repeat style="height: 500px; overflow-y: auto;">
    <div ng-repeat="item in $ctrl.items track by item.id"
         style="height: 50px;">
        <span ng-bind="::item.name"></span>
        <span ng-bind="::item.status"></span>
    </div>
</div>
```

Key requirements:
- Container must have a fixed height with `overflow-y: auto`
- Each item must have a consistent height (or use `vs-repeat` autoresize)
- Combine with one-time bindings (`::`) for maximum performance

### Custom Virtual Scroll Directive

```javascript
angular.module('app.shared')
    .directive('virtualScroll', virtualScroll);

virtualScroll.$inject = ['$window'];

function virtualScroll($window) {
    return {
        restrict: 'A',
        scope: {
            items: '<virtualScroll',
            itemHeight: '<',
            renderItem: '&'
        },
        link: function(scope, element) {
            var container = element[0];
            var visibleCount = Math.ceil(container.clientHeight / scope.itemHeight) + 2;
            var startIndex = 0;

            scope.visibleItems = [];
            scope.topPadding = 0;
            scope.bottomPadding = 0;

            container.addEventListener('scroll', onScroll);

            scope.$on('$destroy', function() {
                container.removeEventListener('scroll', onScroll);
            });

            function onScroll() {
                var newStart = Math.floor(container.scrollTop / scope.itemHeight);
                if (newStart !== startIndex) {
                    startIndex = newStart;
                    scope.$evalAsync(updateVisibleItems);
                }
            }

            function updateVisibleItems() {
                var items = scope.items || [];
                var end = Math.min(startIndex + visibleCount, items.length);
                scope.visibleItems = items.slice(startIndex, end);
                scope.topPadding = startIndex * scope.itemHeight;
                scope.bottomPadding = (items.length - end) * scope.itemHeight;
            }

            scope.$watchCollection('items', updateVisibleItems);
        }
    };
}
```

---

## DOM Recycling Patterns

### ng-repeat with Limited DOM

Instead of rendering all items, maintain a fixed pool of DOM elements and update their content.

```javascript
angular.module('app.shared')
    .component('recyclerView', {
        bindings: {
            items: '<',
            itemHeight: '<',
            containerHeight: '<'
        },
        controller: RecyclerViewController,
        template: '<div class="recycler" style="height:{{$ctrl.containerHeight}}px;overflow-y:auto">' +
                  '  <div style="height:{{$ctrl.totalHeight}}px;position:relative">' +
                  '    <div ng-repeat="vItem in $ctrl.visibleItems track by vItem.$$index"' +
                  '         style="position:absolute;top:{{vItem.$$top}}px;height:{{$ctrl.itemHeight}}px;width:100%">' +
                  '      <span ng-bind="::vItem.name"></span>' +
                  '    </div>' +
                  '  </div>' +
                  '</div>'
    });

function RecyclerViewController($element, $scope) {
    var $ctrl = this;

    $ctrl.$postLink = function() {
        var container = $element[0].querySelector('.recycler');
        container.addEventListener('scroll', function() {
            $scope.$evalAsync(function() {
                updateVisible(container.scrollTop);
            });
        });
    };

    function updateVisible(scrollTop) {
        var start = Math.floor(scrollTop / $ctrl.itemHeight);
        var count = Math.ceil($ctrl.containerHeight / $ctrl.itemHeight) + 1;
        var items = $ctrl.items || [];

        $ctrl.totalHeight = items.length * $ctrl.itemHeight;
        $ctrl.visibleItems = [];

        for (var i = start; i < Math.min(start + count, items.length); i++) {
            var item = angular.copy(items[i]);
            item.$$index = i;
            item.$$top = i * $ctrl.itemHeight;
            $ctrl.visibleItems.push(item);
        }
    }
}
```

---

## ng-if vs ng-show for Conditional Rendering

| Directive | DOM Behavior | Watchers | Use When |
|-----------|-------------|----------|----------|
| `ng-if` | Destroys/creates DOM and scope | Removed when hidden | Content is rarely shown |
| `ng-show` | Hides with CSS (`display:none`) | Always active | Content toggles frequently |
| `ng-switch` | Destroys/creates matching case | Removed when not matched | Mutually exclusive content |

```html
<!-- ng-if: removes watchers when hidden (good for tabs) -->
<div ng-if="$ctrl.activeTab === 'reports'">
    <report-dashboard></report-dashboard>
</div>

<!-- ng-show: keeps watchers active (good for quick toggles) -->
<div ng-show="$ctrl.isTooltipVisible" class="tooltip">
    <span ng-bind="$ctrl.tooltipText"></span>
</div>
```

For tab panels with heavy content, prefer `ng-if` to destroy inactive tabs and their watchers.

---

## Batch DOM Updates

### $evalAsync for Microtask Batching

```javascript
// Instead of multiple digest cycles
items.forEach(function(item) {
    scope.$apply(function() { /* update */ });  // BAD: n digest cycles
});

// Batch into a single digest
scope.$evalAsync(function() {
    items.forEach(function(item) {
        // All updates happen in one digest cycle
    });
});
```

### requestAnimationFrame for Visual Updates

```javascript
// Synchronize DOM reads and writes with the browser's render cycle
function AnimatedDirective($window) {
    return {
        link: function(scope, element) {
            var rafId;

            scope.$watch('progress', function(value) {
                if (rafId) $window.cancelAnimationFrame(rafId);
                rafId = $window.requestAnimationFrame(function() {
                    element[0].style.width = value + '%';
                });
            });

            scope.$on('$destroy', function() {
                if (rafId) $window.cancelAnimationFrame(rafId);
            });
        }
    };
}
```

---

## Production Build Optimizations

### Disable Debug Info

In production, disable AngularJS debug data attached to DOM elements. This reduces memory usage and prevents `angular.element(node).scope()` calls.

```javascript
angular.module('app')
    .config(['$compileProvider', function($compileProvider) {
        $compileProvider.debugInfoEnabled(false);
    }]);
```

To re-enable in the browser console for debugging:

```javascript
angular.reloadWithDebugInfo();
```

### Disable Comment and CSS Class Directives

If you only use element and attribute directives (recommended), disable scanning for comment and CSS class directives:

```javascript
angular.module('app')
    .config(['$compileProvider', function($compileProvider) {
        $compileProvider.commentDirectivesEnabled(false);
        $compileProvider.cssClassDirectivesEnabled(false);
    }]);
```

### Preassign Bindings (1.7+)

Disable pre-assigning bindings to support `$onInit` properly:

```javascript
angular.module('app')
    .config(['$compileProvider', function($compileProvider) {
        $compileProvider.preAssignBindingsEnabled(false);
    }]);
```

---

## Performance Monitoring

### Digest Cycle Dashboard

```javascript
angular.module('app').run(['$rootScope', function($rootScope) {
    if (!window.__NG_PERF__) return;

    var digestCount = 0;
    var totalTime = 0;

    var original$digest = $rootScope.$digest;
    $rootScope.$digest = function() {
        digestCount++;
        var start = performance.now();
        original$digest.apply(this, arguments);
        totalTime += performance.now() - start;

        if (digestCount % 100 === 0) {
            console.table({
                'Total Digests': digestCount,
                'Avg Duration (ms)': (totalTime / digestCount).toFixed(2),
                'Active Watchers': countWatchers()
            });
        }
    };
}]);
```
