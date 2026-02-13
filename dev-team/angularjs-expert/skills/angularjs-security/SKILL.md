---
name: AngularJS Security
description: This skill should be used when the user asks about "AngularJS XSS", "AngularJS security", "$sce", "$sanitize", "ngSanitize", "AngularJS CSRF", "Content Security Policy AngularJS", "template injection", or "AngularJS authentication". It covers AngularJS security hardening including Strict Contextual Escaping (SCE), XSS prevention with $sanitize, CSRF protection, Content Security Policy configuration, template injection prevention, authentication/authorization patterns, and secure HTTP communication.
---

## Strict Contextual Escaping (SCE)

AngularJS uses SCE to automatically escape values in template bindings. SCE is enabled by default and should never be disabled in production.

### How SCE Works

SCE classifies values by context: HTML, CSS, URL, resource URL, and JavaScript. AngularJS refuses to bind untrusted values into sensitive contexts.

```javascript
// SCE contexts
$sce.HTML         // ng-bind-html
$sce.CSS          // ng-style (string values)
$sce.URL          // href, src attributes
$sce.RESOURCE_URL // ng-include, templateUrl
$sce.JS           // onclick (should never be used)
```

### Safe Rendering of User Content

Never use `$sce.trustAsHtml()` on user-provided content. Use `$sanitize` instead.

```javascript
// DANGEROUS: trusts raw user input as safe HTML
$ctrl.content = $sce.trustAsHtml(userInput);

// SAFE: sanitize user input, stripping dangerous tags and attributes
angular.module('app', ['ngSanitize']);

// In the template, ng-bind-html automatically runs $sanitize
// <div ng-bind-html="$ctrl.content"></div>
$ctrl.content = userInput; // $sanitize runs automatically via ng-bind-html
```

### When trustAsHtml Is Acceptable

Only use `$sce.trustAsHtml()` for content from your own server that you fully control and has already been sanitized server-side:

```javascript
function ContentService($http, $sce) {
    return {
        getRenderedMarkdown: function(articleId) {
            return $http.get('/api/articles/' + articleId + '/rendered')
                .then(function(response) {
                    // Server-rendered and sanitized markdown — safe to trust
                    return $sce.trustAsHtml(response.data.html);
                });
        }
    };
}
```

---

## XSS Prevention

### Template Binding Rules

| Binding | Escapes | Safe for User Input | Use Case |
|---------|---------|---------------------|----------|
| `{{ expression }}` | Yes (HTML-encoded) | Yes | Text content |
| `ng-bind` | Yes (HTML-encoded) | Yes | Text content (preferred) |
| `ng-bind-html` | Yes (via $sanitize) | Yes (with ngSanitize) | Rich text from users |
| `$sce.trustAsHtml()` | No | No | Pre-sanitized server content only |
| `$compile(userInput)` | No | Never | Dangerous — avoid |

### Use ng-bind Over Interpolation

Prefer `ng-bind` over `{{ }}` interpolation. `ng-bind` does not flash unrendered template syntax on slow page loads, and it makes intent explicit.

```html
<!-- Good: no flash of {{ }} on load, explicit binding -->
<span ng-bind="$ctrl.userName"></span>

<!-- Acceptable but may flash on load -->
<span>{{ $ctrl.userName }}</span>

<!-- DANGEROUS: user content rendered as HTML without sanitization -->
<div ng-bind-html-unsafe="$ctrl.userBio"></div>  <!-- deprecated, removed -->
```

### Sanitization Allowlist Configuration

Customize the `$sanitize` allowlist if you need specific HTML tags:

```javascript
angular.module('app')
    .config(['$sanitizeProvider', function($sanitizeProvider) {
        // Add custom allowed tags (use sparingly)
        $sanitizeProvider.addValidElements(['video', 'source']);
        $sanitizeProvider.addValidAttrs(['controls', 'autoplay']);
    }]);
```

### Preventing Template Injection

Never compile user-provided strings as AngularJS templates.

```javascript
// DANGEROUS: server-side template injection
// If the server returns {{ constructor.constructor('alert(1)')() }}
// it will execute as JavaScript
$http.get('/api/template').then(function(response) {
    var compiled = $compile(response.data)(scope);  // XSS via template injection
    element.append(compiled);
});

// SAFE: use $templateCache with known templates
$templateRequest('features/users/profile.html').then(function(template) {
    var compiled = $compile(template)(scope);
    element.append(compiled);
});
```

---

## CSRF Protection

### Token Handling with $http

AngularJS can automatically read CSRF tokens from cookies and attach them to outgoing requests.

```javascript
angular.module('app')
    .config(['$httpProvider', function($httpProvider) {
        // Tell AngularJS which cookie contains the CSRF token
        $httpProvider.defaults.xsrfCookieName = 'XSRF-TOKEN';
        // Tell AngularJS which header to send it in
        $httpProvider.defaults.xsrfHeaderName = 'X-XSRF-TOKEN';
    }]);
```

### Custom CSRF Interceptor

For APIs that use a non-cookie CSRF mechanism (e.g., meta tag or custom endpoint):

```javascript
angular.module('app.core')
    .factory('csrfInterceptor', csrfInterceptor);

function csrfInterceptor() {
    var csrfToken = document.querySelector('meta[name="csrf-token"]');

    return {
        request: function(config) {
            if (csrfToken && isStateChangingMethod(config.method)) {
                config.headers['X-CSRF-Token'] = csrfToken.getAttribute('content');
            }
            return config;
        }
    };

    function isStateChangingMethod(method) {
        return ['POST', 'PUT', 'PATCH', 'DELETE'].indexOf(method.toUpperCase()) > -1;
    }
}
```

---

## Content Security Policy (CSP)

### AngularJS CSP Mode

AngularJS uses `Function()` constructor and `eval()` by default for template expression evaluation. Enable CSP-compatible mode to avoid these:

```html
<!-- Enable CSP mode on the root element -->
<html ng-app="app" ng-csp>
```

Or selectively:

```html
<!-- Disable eval but allow inline styles -->
<html ng-app="app" ng-csp="no-unsafe-eval">

<!-- Disable both eval and inline styles -->
<html ng-app="app" ng-csp="no-unsafe-eval;no-inline-style">
```

### Recommended CSP Header

```
Content-Security-Policy:
    default-src 'self';
    script-src 'self';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self';
    connect-src 'self' https://api.example.com;
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self';
```

When `ng-csp` is enabled, AngularJS uses a CSP-safe expression evaluator. Note: this has a slight performance cost compared to the default `eval`-based evaluator.

### CSP Violation Reporting

```
Content-Security-Policy-Report-Only:
    default-src 'self';
    report-uri /api/csp-violations;
```

```javascript
// Server endpoint to collect CSP violation reports
app.post('/api/csp-violations', function(req, res) {
    console.log('CSP Violation:', req.body);
    res.status(204).end();
});
```

---

## Authentication and Authorization

### Token-Based Authentication Pattern

```javascript
// core/services/auth.service.js
function AuthService($http, $window, $q, API_URL) {
    var currentUser = null;
    var TOKEN_KEY = 'auth_token';

    return {
        login: login,
        logout: logout,
        isAuthenticated: isAuthenticated,
        getCurrentUser: getCurrentUser,
        getToken: getToken
    };

    function login(credentials) {
        return $http.post(API_URL + '/auth/login', credentials)
            .then(function(response) {
                $window.sessionStorage.setItem(TOKEN_KEY, response.data.token);
                currentUser = response.data.user;
                return currentUser;
            });
    }

    function logout() {
        $window.sessionStorage.removeItem(TOKEN_KEY);
        currentUser = null;
    }

    function isAuthenticated() {
        return !!getToken();
    }

    function getCurrentUser() {
        if (currentUser) return $q.resolve(currentUser);
        if (!isAuthenticated()) return $q.reject('Not authenticated');

        return $http.get(API_URL + '/auth/me')
            .then(function(response) {
                currentUser = response.data;
                return currentUser;
            });
    }

    function getToken() {
        return $window.sessionStorage.getItem(TOKEN_KEY);
    }
}
```

### Route-Level Authorization

```javascript
// State definition with access control
$stateProvider.state('admin', {
    url: '/admin',
    component: 'adminDashboard',
    data: {
        requiresAuth: true,
        roles: ['admin']
    },
    resolve: {
        currentUser: ['AuthService', function(AuthService) {
            return AuthService.getCurrentUser();
        }]
    }
});

// Transition hook for authorization
$transitions.onBefore({}, function(transition) {
    var toState = transition.to();

    if (toState.data && toState.data.requiresAuth) {
        var AuthService = transition.injector().get('AuthService');

        if (!AuthService.isAuthenticated()) {
            return transition.router.stateService.target('login');
        }

        if (toState.data.roles) {
            return AuthService.getCurrentUser().then(function(user) {
                var hasRole = toState.data.roles.some(function(role) {
                    return user.roles.indexOf(role) > -1;
                });
                if (!hasRole) {
                    return transition.router.stateService.target('forbidden');
                }
            });
        }
    }
});
```

---

## Secure HTTP Communication

### HTTPS Enforcement

```javascript
// Interceptor to block non-HTTPS requests in production
angular.module('app.core')
    .factory('httpsInterceptor', httpsInterceptor);

function httpsInterceptor($q, APP_CONFIG) {
    return {
        request: function(config) {
            if (APP_CONFIG.enforceHttps && config.url.indexOf('http://') === 0) {
                config.url = config.url.replace('http://', 'https://');
            }
            return config;
        }
    };
}
```

### Sensitive Data Handling

```javascript
// Never log sensitive data
function AuthInterceptor($log) {
    return {
        request: function(config) {
            // Log the URL but never the body or headers containing tokens
            $log.debug('HTTP Request:', config.method, config.url);
            return config;
        },
        response: function(response) {
            // Never log response bodies that may contain tokens or PII
            $log.debug('HTTP Response:', response.status, response.config.url);
            return response;
        }
    };
}
```

---

## URL Sanitization

AngularJS sanitizes URLs in `href` and `src` attributes by default. Only `http`, `https`, `ftp`, `mailto`, `tel`, and `data:image/*` protocols are allowed.

### Custom Protocol Allowlist

```javascript
angular.module('app')
    .config(['$compileProvider', function($compileProvider) {
        // Add custom protocol (e.g., for desktop app deep links)
        $compileProvider.aHrefSanitizationTrustedUrlList(
            /^\s*(https?|ftp|mailto|tel|myapp):/
        );

        // For image sources
        $compileProvider.imgSrcSanitizationTrustedUrlList(
            /^\s*(https?|data:image\/(png|jpg|jpeg|gif|svg\+xml))/
        );
    }]);
```

---

## References

- **[XSS Defense Patterns](references/xss-defense-patterns.md)** — Deep dive into client-side sanitization, DOM-based XSS vectors, and defense-in-depth strategies specific to AngularJS.
- **[Authentication Patterns](references/authentication-patterns.md)** — JWT handling, refresh token rotation, session management, and OAuth integration patterns for AngularJS single-page applications.
