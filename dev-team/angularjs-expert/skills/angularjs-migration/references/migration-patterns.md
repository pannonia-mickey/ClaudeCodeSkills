# AngularJS Migration Patterns

Detailed reference covering service migration, routing migration, state management transition, and handling third-party AngularJS libraries during migration.

---

## Service Migration Pattern

### Step 1: Create Angular Service with Same Interface

```typescript
// Angular version of UserService
@Injectable({ providedIn: 'root' })
export class UserService {
    constructor(private http: HttpClient) {}

    getAll(): Observable<User[]> {
        return this.http.get<User[]>(`${environment.apiUrl}/users`);
    }

    getById(id: number): Observable<User> {
        return this.http.get<User>(`${environment.apiUrl}/users/${id}`);
    }

    create(user: Partial<User>): Observable<User> {
        return this.http.post<User>(`${environment.apiUrl}/users`, user);
    }

    update(id: number, user: Partial<User>): Observable<User> {
        return this.http.put<User>(`${environment.apiUrl}/users/${id}`, user);
    }
}
```

### Step 2: Downgrade for AngularJS Consumption

```typescript
// Provide Angular service to AngularJS
angular.module('app.core')
    .factory('UserService', downgradeInjectable(UserService) as any);
```

### Step 3: Migrate Consumers

Gradually update AngularJS components that use the service. The service interface is the same, but now returns Observables instead of promises. Use `.toPromise()` as a bridge:

```typescript
// Transitional: AngularJS component consuming Angular Observable service
$ctrl.$onInit = function() {
    UserService.getAll().toPromise().then(function(users) {
        $ctrl.users = users;
    });
};
```

### Step 4: Remove AngularJS Wrapper

Once all consumers are migrated to Angular, remove the `downgradeInjectable` wrapper.

---

## Routing Migration

### Parallel Routing Strategy

Run both AngularJS ui-router and Angular Router simultaneously during migration:

```typescript
// Angular module with routing
@NgModule({
    imports: [
        RouterModule.forRoot([
            // Migrated routes go here
            { path: 'dashboard', component: DashboardComponent },
            { path: 'settings', component: SettingsComponent },
        ], {
            // Let Angular Router handle only known routes
            // Unknown routes fall through to AngularJS ui-router
            initialNavigation: 'enabledBlocking'
        })
    ]
})
export class AppRoutingModule {}
```

```javascript
// AngularJS ui-router — handle remaining routes
$urlRouterProvider.otherwise(function($injector) {
    // If Angular Router didn't handle it, ui-router gets it
    var $state = $injector.get('$state');
    $state.go('legacy-home');
});
```

### Route Migration Order

1. Migrate leaf routes first (no child routes)
2. Migrate parent routes, converting child AngularJS routes to Angular child routes
3. Migrate the root/layout route last

---

## State Management Transition

### From $rootScope Events to RxJS

```typescript
// AngularJS pattern: $rootScope event bus
$rootScope.$broadcast('user:updated', updatedUser);
$scope.$on('user:updated', function(event, user) { /* ... */ });

// Angular pattern: shared service with Subject
@Injectable({ providedIn: 'root' })
export class EventBusService {
    private userUpdated$ = new Subject<User>();

    emitUserUpdated(user: User): void {
        this.userUpdated$.next(user);
    }

    onUserUpdated(): Observable<User> {
        return this.userUpdated$.asObservable();
    }
}

// Transitional bridge: downgrade for AngularJS
angular.module('app.core')
    .factory('EventBusService', downgradeInjectable(EventBusService));

// AngularJS component using the bridge
EventBusService.onUserUpdated().subscribe(function(user) {
    $ctrl.user = user;
    $scope.$apply(); // Trigger AngularJS digest
});
```

---

## Third-Party Library Handling

### Libraries with Angular Alternatives

| AngularJS Library | Angular Replacement |
|------------------|-------------------|
| `ui-router` | `@angular/router` |
| `angular-translate` | `@ngx-translate/core` |
| `angular-ui-bootstrap` | `ng-bootstrap` or `@angular/material` |
| `restangular` | `HttpClient` |
| `angular-moment` | `date-fns` + pipes |
| `angular-file-upload` | Native `HttpClient` + `FormData` |
| `angular-toastr` | `ngx-toastr` |

### Libraries Without Angular Alternatives

For AngularJS-specific libraries without Angular equivalents, wrap them in an Angular service:

```typescript
@Injectable({ providedIn: 'root' })
export class LegacyChartService {
    constructor(@Inject(UpgradeModule) private upgrade: UpgradeModule) {}

    getService(): any {
        return this.upgrade.$injector.get('legacyChartService');
    }
}
```

Or replace with framework-agnostic alternatives (e.g., replace `angular-chart.js` with `Chart.js` directly).

---

## Migration Metrics

Track these metrics to measure migration progress:

| Metric | How to Measure | Target |
|--------|---------------|--------|
| % components migrated | Count Angular vs AngularJS components | 100% |
| % services migrated | Count Angular vs AngularJS services | 100% |
| % routes migrated | Count Angular Router vs ui-router states | 100% |
| Bundle size | AngularJS bundle size trending to 0 | 0 KB |
| Test coverage | Angular TestBed tests vs Karma/Jasmine | 100% Angular |
| Downgraded components | Count `downgradeComponent` calls | 0 |
| Upgraded services | Count `downgradeInjectable` calls | 0 |

---

## Common Migration Pitfalls

### 1. Change Detection Boundaries

AngularJS uses digest cycles; Angular uses Zone.js. When crossing boundaries:

```typescript
// Angular component receiving events from AngularJS
constructor(private ngZone: NgZone) {}

ngOnInit() {
    // AngularJS callback runs outside Angular zone
    this.legacyService.onChange((data) => {
        this.ngZone.run(() => {
            // Now Angular knows about the change
            this.data = data;
        });
    });
}
```

### 2. Digest/Zone Interplay

Hybrid apps can suffer from excessive change detection. AngularJS `$apply` triggers Angular change detection and vice versa.

Mitigation:
- Use `OnPush` change detection in Angular components
- Avoid frequent cross-boundary calls
- Batch updates where possible

### 3. Dependency Injection Collision

Both frameworks have DI systems. Ensure:
- AngularJS service names don't collide with Angular injection tokens
- Use `InjectionToken` instead of string tokens in Angular
- Explicitly document which DI system owns each service
