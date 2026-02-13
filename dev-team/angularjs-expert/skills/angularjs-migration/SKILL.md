---
name: AngularJS Migration
description: This skill should be used when the user asks about "AngularJS to Angular", "ngUpgrade", "AngularJS migration", "hybrid app AngularJS", "upgrade AngularJS", or "migrate from AngularJS". It covers migration strategies from AngularJS (1.x) to modern Angular (2+), including preparation steps, the ngUpgrade hybrid approach, component-by-component migration, TypeScript adoption, and the strangler fig pattern for incremental migration without rewriting.
---

## Migration Strategy Overview

AngularJS reached end of life in December 2021. Migration to modern Angular (or another framework) is essential for long-term maintainability and security.

### Migration Approaches

| Approach | Effort | Risk | Best For |
|----------|--------|------|----------|
| **Big Bang Rewrite** | Very High | Very High | Small apps (< 50 components) |
| **ngUpgrade Hybrid** | High | Medium | Medium apps, gradual migration |
| **Strangler Fig** | Medium | Low | Large apps, team-at-a-time migration |
| **Micro-Frontend** | Medium-High | Low | Large apps with independent teams |

### Recommended: Strangler Fig + ngUpgrade

For most applications, combine the Strangler Fig pattern with ngUpgrade:
1. Prepare the AngularJS codebase (component architecture, TypeScript)
2. Set up a hybrid Angular/AngularJS shell
3. Migrate feature by feature, routing Angular components alongside AngularJS ones
4. Remove AngularJS once all features are migrated

---

## Phase 1: Prepare the AngularJS Codebase

Before touching Angular, prepare the AngularJS app to minimize migration friction.

### Step 1: Adopt Component Architecture

Convert all controllers + templates to `.component()` definitions. Components map directly to Angular components.

```javascript
// Before: controller + route template
$stateProvider.state('users', {
    url: '/users',
    templateUrl: 'users/user-list.html',
    controller: 'UserListController',
    controllerAs: 'vm'
});

// After: component + route
$stateProvider.state('users', {
    url: '/users',
    component: 'userList'
});

angular.module('app.users')
    .component('userList', {
        templateUrl: 'users/user-list.html',
        controller: UserListController,
        bindings: {
            users: '<'  // fed by route resolve
        }
    });
```

### Step 2: Switch to One-Way Bindings

Replace all `=` (two-way) bindings with `<` (one-way) input and `&` (callback) output:

```javascript
// Before
bindings: {
    user: '=',      // two-way
    items: '='       // two-way
}

// After
bindings: {
    user: '<',       // one-way input
    items: '<',      // one-way input
    onUpdate: '&'    // output callback
}
```

### Step 3: Adopt TypeScript

Incrementally convert `.js` files to `.ts`. Start with services and models, then components.

```typescript
// user.model.ts
export interface User {
    id: number;
    name: string;
    email: string;
    roles: string[];
}

// user.service.ts
export class UserService {
    static $inject = ['$http', 'API_URL'];

    constructor(
        private $http: angular.IHttpService,
        private API_URL: string
    ) {}

    getAll(): angular.IPromise<User[]> {
        return this.$http.get<User[]>(`${this.API_URL}/users`)
            .then(response => response.data);
    }

    getById(id: number): angular.IPromise<User> {
        return this.$http.get<User>(`${this.API_URL}/users/${id}`)
            .then(response => response.data);
    }
}

angular.module('app.core')
    .service('UserService', UserService);
```

```json
// tsconfig.json for AngularJS + TypeScript
{
    "compilerOptions": {
        "target": "es5",
        "module": "commonjs",
        "strict": true,
        "esModuleInterop": true,
        "types": ["angular", "angular-mocks", "jasmine"],
        "outDir": "./dist",
        "rootDir": "./src"
    },
    "include": ["src/**/*"]
}
```

### Step 4: Adopt Module Bundler

Switch from script concatenation (Gulp/Grunt) to Webpack:

```javascript
// webpack.config.js
module.exports = {
    entry: './src/app/app.module.ts',
    output: {
        path: path.resolve(__dirname, 'dist'),
        filename: 'bundle.js'
    },
    resolve: {
        extensions: ['.ts', '.js']
    },
    module: {
        rules: [
            { test: /\.ts$/, use: 'ts-loader' },
            { test: /\.html$/, use: 'html-loader' }
        ]
    }
};
```

### Step 5: Use ES Module Imports

Convert from IIFE + global `angular.module()` to ES module imports:

```typescript
// Before (IIFE)
(function() {
    angular.module('app.users')
        .component('userList', { /* ... */ });
})();

// After (ES module)
import * as angular from 'angular';
import { UserListController } from './user-list.controller';

angular.module('app.users')
    .component('userList', {
        templateUrl: require('./user-list.html'),
        controller: UserListController,
        bindings: { users: '<' }
    });
```

---

## Phase 2: Set Up ngUpgrade Hybrid

### Bootstrap the Hybrid App

```typescript
// main.ts — Angular entry point
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { UpgradeModule } from '@angular/upgrade/static';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

@NgModule({
    imports: [
        BrowserModule,
        UpgradeModule
    ]
})
export class AppModule {
    constructor(private upgrade: UpgradeModule) {}

    ngDoBootstrap() {
        // Bootstrap AngularJS inside Angular
        this.upgrade.bootstrap(document.body, ['app'], { strictDi: true });
    }
}

platformBrowserDynamic().bootstrapModule(AppModule);
```

### Downgrade Angular Components for Use in AngularJS

```typescript
// Expose an Angular component to AngularJS templates
import { downgradeComponent } from '@angular/upgrade/static';
import { NewUserProfileComponent } from './features/users/user-profile.component';

angular.module('app.users')
    .directive('newUserProfile', downgradeComponent({
        component: NewUserProfileComponent,
        inputs: ['userId'],
        outputs: ['userUpdated']
    }) as angular.IDirectiveFactory);
```

Usage in AngularJS template:

```html
<!-- AngularJS template using the Angular component -->
<new-user-profile
    user-id="$ctrl.selectedUserId"
    on-user-updated="$ctrl.handleUserUpdate($event)">
</new-user-profile>
```

### Upgrade AngularJS Services for Use in Angular

```typescript
// Make an AngularJS service available to Angular components
import { InjectionToken } from '@angular/core';

export const UserServiceToken = new InjectionToken<any>('UserService');

@NgModule({
    providers: [{
        provide: UserServiceToken,
        useFactory: (injector: any) => injector.get('UserService'),
        deps: ['$injector']
    }]
})
export class AppModule {}
```

Usage in Angular component:

```typescript
@Component({ /* ... */ })
export class NewDashboardComponent {
    constructor(@Inject(UserServiceToken) private userService: any) {}

    ngOnInit() {
        this.userService.getAll().then((users: User[]) => {
            this.users = users;
        });
    }
}
```

---

## Phase 3: Migrate Feature by Feature

### Migration Order

1. **Leaf components first**: Components with no AngularJS children
2. **Shared services**: Migrate services to Angular, provide to both sides via upgrade/downgrade
3. **Feature modules**: Migrate entire feature modules (components + services + routes)
4. **Core services**: Auth, HTTP interceptors, error handling
5. **Root component and routing**: Switch to Angular Router last

### Component Migration Checklist

For each component being migrated:

- [ ] Create Angular component with equivalent template
- [ ] Replace `ng-repeat` with `*ngFor`
- [ ] Replace `ng-if` / `ng-show` with `*ngIf`
- [ ] Replace `ng-model` with `[(ngModel)]`
- [ ] Replace `ng-click` with `(click)`
- [ ] Replace `ng-class` with `[ngClass]`
- [ ] Replace `ng-bind` with `{{ }}` or `[innerText]`
- [ ] Replace `$http` with `HttpClient`
- [ ] Replace promises with Observables (or use `.toPromise()` transitionally)
- [ ] Replace `$onInit` with `ngOnInit`
- [ ] Replace `$onDestroy` with `ngOnDestroy`
- [ ] Replace `$onChanges` with `ngOnChanges`
- [ ] Add TypeScript types for all inputs/outputs
- [ ] Downgrade the new component for any remaining AngularJS parents
- [ ] Update tests to use Angular TestBed
- [ ] Remove the old AngularJS component

### Template Mapping Reference

| AngularJS | Angular | Notes |
|-----------|---------|-------|
| `ng-repeat="item in items"` | `*ngFor="let item of items"` | `trackBy` function instead of `track by` |
| `ng-if="condition"` | `*ngIf="condition"` | |
| `ng-show="condition"` | `[hidden]="!condition"` | Or use `*ngIf` |
| `ng-model="value"` | `[(ngModel)]="value"` | Import `FormsModule` |
| `ng-click="handler()"` | `(click)="handler()"` | |
| `ng-class="{ active: isActive }"` | `[ngClass]="{ active: isActive }"` | |
| `ng-style="{ color: textColor }"` | `[ngStyle]="{ color: textColor }"` | |
| `ng-bind="value"` | `{{ value }}` | Or `[innerText]="value"` |
| `ng-bind-html="html"` | `[innerHTML]="html"` | Use `DomSanitizer` for trusted HTML |
| `ng-submit="submit()"` | `(ngSubmit)="submit()"` | Import `FormsModule` |
| `ng-change="onChange()"` | `(ngModelChange)="onChange()"` | |
| `ng-disabled="isDisabled"` | `[disabled]="isDisabled"` | |
| `ng-href="{{ url }}"` | `[href]="url"` | |
| `{{ value \| currency }}` | `{{ value \| currency }}` | Import Angular pipe |

---

## Phase 4: Remove AngularJS

### Final Steps

1. Remove `@angular/upgrade` from dependencies
2. Remove all `downgradeComponent` and `upgradeModule` references
3. Replace the hybrid bootstrap with standard Angular bootstrap
4. Remove `angular`, `angular-mocks`, and all AngularJS plugins from `package.json`
5. Remove Karma configuration (switch to Jest or Karma with Angular CLI)
6. Update CI pipeline to use Angular CLI

### Verification

- [ ] No `angular.module()` calls remain
- [ ] No `$scope`, `$http`, `$state` references remain
- [ ] No `ng-` prefixed attributes in templates
- [ ] All routes use Angular Router
- [ ] All tests use TestBed
- [ ] Build size decreased (no AngularJS bundle)

---

## References

- **[Migration Patterns](references/migration-patterns.md)** — Detailed patterns for service migration, routing migration, state management transition, and handling third-party AngularJS libraries during migration.
- **[Hybrid App Troubleshooting](references/hybrid-troubleshooting.md)** — Common issues with ngUpgrade hybrid apps including change detection, routing conflicts, digest/zone interplay, and dependency injection across boundaries.
