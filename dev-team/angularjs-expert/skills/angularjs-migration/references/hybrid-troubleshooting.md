# Hybrid App Troubleshooting

Common issues with ngUpgrade hybrid applications and their solutions, covering change detection, routing conflicts, digest/zone interplay, and dependency injection across boundaries.

---

## Change Detection Issues

### Problem: Angular Component Not Updating

**Symptom**: An Angular component receives data from AngularJS but the view does not update.

**Cause**: The AngularJS callback executes outside Angular's NgZone.

**Solution**:

```typescript
@Component({
    selector: 'app-user-detail',
    template: `<div>{{ user?.name }}</div>`,
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserDetailComponent implements OnInit {
    user: User | null = null;

    constructor(
        private ngZone: NgZone,
        private cdr: ChangeDetectorRef,
        @Inject(UserServiceToken) private userService: any
    ) {}

    ngOnInit() {
        // AngularJS service returns a promise that resolves outside NgZone
        this.ngZone.run(() => {
            this.userService.getById(this.userId).then((user: User) => {
                this.user = user;
                this.cdr.markForCheck(); // Required for OnPush
            });
        });
    }
}
```

### Problem: AngularJS Not Updating After Angular Event

**Symptom**: An AngularJS component does not reflect changes triggered by an Angular component.

**Cause**: Angular's change detection does not trigger AngularJS's `$digest`.

**Solution**:

```typescript
// In the Angular component's output handler
@Output() userSelected = new EventEmitter<User>();

selectUser(user: User) {
    this.userSelected.emit(user);
}
```

```javascript
// In the AngularJS parent, the downgraded component output triggers $digest
// automatically via ngUpgrade's event binding. If it doesn't:
$scope.$applyAsync(function() {
    $ctrl.selectedUser = event.user;
});
```

---

## Routing Conflicts

### Problem: Both Routers Fight Over URL Changes

**Symptom**: Clicking a link causes a blank page or incorrect component loading.

**Cause**: Both Angular Router and ui-router respond to the same URL change.

**Solution**: Use `UrlHandlingStrategy` to partition URLs:

```typescript
// Only let Angular Router handle specific URL prefixes
@Injectable()
export class HybridUrlHandlingStrategy implements UrlHandlingStrategy {
    // List of URL prefixes that Angular Router should handle
    private angularRoutes = ['/dashboard', '/settings', '/profile'];

    shouldProcessUrl(url: UrlTree): boolean {
        return this.angularRoutes.some(route =>
            url.toString().startsWith(route)
        );
    }

    extract(url: UrlTree): UrlTree { return url; }
    merge(url: UrlTree, whole: UrlTree): UrlTree { return url; }
}

@NgModule({
    providers: [{
        provide: UrlHandlingStrategy,
        useClass: HybridUrlHandlingStrategy
    }]
})
export class AppModule {}
```

### Problem: Browser Back Button Breaks

**Symptom**: Pressing browser back navigates to the wrong state or shows blank content.

**Solution**: Ensure only one router manages history at a time. For the partitioned approach above, each router only responds to its own URL prefix, so back/forward works correctly within each partition.

---

## Digest/Zone Performance Issues

### Problem: Slow Page After Adding Hybrid Components

**Symptom**: Adding an Angular component to an AngularJS page makes everything slower.

**Cause**: Every AngularJS `$digest` triggers Angular change detection, and vice versa. With many hybrid components, this creates a cascade.

**Solutions**:

1. **Use OnPush change detection on all Angular components in the hybrid app**:

```typescript
@Component({
    changeDetection: ChangeDetectionStrategy.OnPush
})
```

2. **Detach from Angular change detection for static content**:

```typescript
constructor(private cdr: ChangeDetectorRef) {}

ngOnInit() {
    // Load data once, then detach
    this.loadData().subscribe(data => {
        this.data = data;
        this.cdr.detectChanges();
        this.cdr.detach(); // Stop automatic change detection
    });
}
```

3. **Minimize cross-boundary communication**. Batch updates instead of emitting events individually.

---

## Dependency Injection Issues

### Problem: "Cannot find AngularJS service" in Angular Component

**Symptom**: `NullInjectorError` when trying to use an AngularJS service in Angular.

**Cause**: The AngularJS service is not registered as an Angular provider.

**Solution**:

```typescript
// Register the AngularJS service as an Angular provider
@NgModule({
    providers: [{
        provide: 'LegacyUserService',
        useFactory: ($injector: any) => $injector.get('UserService'),
        deps: ['$injector']
    }]
})
export class AppModule {}
```

### Problem: Circular Dependency Between Angular and AngularJS Services

**Symptom**: `Error: NG0200: Circular dependency in DI detected`.

**Cause**: Angular service A depends on downgraded AngularJS service B, which depends on upgraded Angular service A.

**Solution**: Break the cycle with lazy injection:

```typescript
@Injectable()
export class AngularServiceA {
    constructor(private injector: Injector) {}

    doSomething() {
        // Lazy-load to break circular dependency
        const serviceB = this.injector.get('LegacyServiceB');
        return serviceB.process();
    }
}
```

---

## Template Issues

### Problem: AngularJS Directives Not Working Inside Angular Components

**Symptom**: `ng-click`, `ng-if`, and other AngularJS directives are ignored inside Angular component templates.

**Cause**: Angular components use Angular's template compiler, which does not understand AngularJS directives.

**Solution**: You cannot use AngularJS directives in Angular templates. Instead:
- Use Angular equivalents (`(click)`, `*ngIf`, etc.)
- If you need AngularJS functionality, keep it in an AngularJS component and use `downgradeComponent` to embed it

### Problem: Angular Components Not Rendering in AngularJS Templates

**Symptom**: The Angular component tag appears in the DOM but nothing renders.

**Cause**: The component was not properly downgraded.

**Checklist**:
1. The component is declared in an Angular `@NgModule`'s `declarations` array
2. The component is added to `entryComponents` (Angular < 9) or the module is compiled
3. `downgradeComponent` is called with the correct component class
4. The directive name in AngularJS follows camelCase → kebab-case convention
5. Input/output bindings use the correct attribute naming (camelCase → kebab-case)

```typescript
// Angular component
@Component({ selector: 'app-user-card', /* ... */ })
export class UserCardComponent {
    @Input() userId!: number;
    @Output() cardClicked = new EventEmitter<void>();
}

// Downgrade
angular.module('app')
    .directive('appUserCard', downgradeComponent({
        component: UserCardComponent
    }) as angular.IDirectiveFactory);
```

```html
<!-- AngularJS template — use kebab-case -->
<app-user-card
    user-id="$ctrl.selectedUserId"
    on-card-clicked="$ctrl.handleClick()">
</app-user-card>
```

---

## Testing Hybrid Components

### Testing Downgraded Angular Components in AngularJS Context

```typescript
describe('Downgraded UserCard in AngularJS', function() {
    beforeEach(angular.mock.module('app'));

    it('should render the Angular component', inject(function($compile, $rootScope) {
        var scope = $rootScope.$new();
        scope.userId = 42;

        var element = $compile(
            '<app-user-card user-id="userId"></app-user-card>'
        )(scope);
        scope.$digest();

        // The Angular component should have rendered
        expect(element.find('.user-card').length).toBeGreaterThan(0);
    }));
});
```

### Testing Upgraded AngularJS Services in Angular Context

```typescript
describe('Upgraded UserService', () => {
    let service: any;

    beforeEach(() => {
        TestBed.configureTestingModule({
            imports: [UpgradeModule],
            providers: [{
                provide: 'UserService',
                useFactory: (injector: any) => injector.get('UserService'),
                deps: ['$injector']
            }]
        });

        service = TestBed.inject('UserService' as any);
    });

    it('should return users', async () => {
        const users = await service.getAll();
        expect(users.length).toBeGreaterThan(0);
    });
});
```
