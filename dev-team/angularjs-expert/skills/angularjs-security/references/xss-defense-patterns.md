# XSS Defense Patterns for AngularJS

Comprehensive reference covering client-side sanitization, DOM-based XSS vectors, AngularJS-specific attack surfaces, and defense-in-depth strategies.

---

## AngularJS-Specific XSS Vectors

### 1. Template Injection via Server-Side Rendering

When server-side templates include user input that contains AngularJS expressions, the client-side AngularJS engine evaluates them.

**Attack**: User submits `{{ constructor.constructor('alert(document.cookie)')() }}` as their username. The server renders it into the HTML. AngularJS evaluates the expression.

**Defense**:
- Always HTML-encode user input on the server before rendering into pages
- Use `ng-bind` instead of `{{ }}` for dynamic content
- Never render user input inside `ng-init` or other expression-evaluating directives

```html
<!-- Vulnerable: server renders user.name into template -->
<span>Welcome, {{ user.name }}</span>

<!-- Safe: server renders static binding, AngularJS evaluates from model -->
<span ng-bind="$ctrl.user.name"></span>
```

### 2. $compile with User Content

`$compile` converts strings into executable AngularJS templates. Never pass user content to `$compile`.

```javascript
// DANGEROUS
function renderUserContent(content) {
    var compiled = $compile('<div>' + content + '</div>')(scope);
    element.append(compiled);
}

// SAFE: use ng-bind-html with $sanitize
$ctrl.safeContent = content;
// <div ng-bind-html="$ctrl.safeContent"></div>
```

### 3. ng-include with Dynamic Paths

If the template URL is user-controllable, an attacker can load arbitrary templates.

```html
<!-- DANGEROUS: user-controlled template path -->
<div ng-include="$ctrl.templateUrl"></div>

<!-- SAFE: whitelist allowed templates -->
```

```javascript
var ALLOWED_TEMPLATES = {
    'profile': 'features/users/profile.html',
    'settings': 'features/users/settings.html'
};

$ctrl.$onInit = function() {
    $ctrl.templateUrl = ALLOWED_TEMPLATES[$ctrl.viewName] || ALLOWED_TEMPLATES.profile;
};
```

### 4. href/src Attribute Injection

```html
<!-- DANGEROUS: user-controlled URL can inject javascript: protocol -->
<a ng-href="{{ $ctrl.userProvidedUrl }}">Link</a>

<!-- AngularJS sanitizes this by default, but verify the allowlist -->
```

AngularJS blocks `javascript:` URLs by default, but always verify the sanitization regex has not been loosened with `$compileProvider.aHrefSanitizationTrustedUrlList()`.

---

## DOM-Based XSS Prevention

### jQuery Integration Risks

AngularJS applications that use jQuery must be careful with `.html()`, `.append()`, and `.after()` methods in directive link functions.

```javascript
// DANGEROUS: injects raw HTML from the model
link: function(scope, element) {
    scope.$watch('content', function(value) {
        element.html(value);  // XSS if value is user-controlled
    });
}

// SAFE: use text() for plain text, or $sanitize for HTML
link: function(scope, element, attrs, ctrl, $transclude) {
    scope.$watch('content', function(value) {
        element.text(value);  // Safe: HTML-encoded
    });
}

// SAFE: sanitize before inserting HTML
link: function(scope, element) {
    var $sanitize = $injector.get('$sanitize');
    scope.$watch('content', function(value) {
        element.html($sanitize(value));
    });
}
```

### Third-Party Library Integration

When integrating third-party libraries that manipulate the DOM (e.g., rich text editors, charting libraries), ensure they do not render unsanitized user content:

```javascript
// Sanitize before passing to third-party library
$ctrl.$onChanges = function(changes) {
    if (changes.content) {
        var clean = $sanitize(changes.content.currentValue);
        thirdPartyEditor.setContent(clean);
    }
};
```

---

## Defense-in-Depth Checklist

### Layer 1: Server-Side

- HTML-encode all user input before rendering into pages
- Implement Content Security Policy headers
- Set `HttpOnly` and `Secure` flags on session cookies
- Validate and sanitize user input on the server

### Layer 2: AngularJS Framework

- Keep `ng-csp` enabled in production
- Never disable SCE (`$sceProvider.enabled(false)`)
- Use `ngSanitize` module for any HTML rendering
- Avoid `$compile` with dynamic content
- Use `ng-bind` over `{{ }}` interpolation

### Layer 3: Application Code

- Never use `$sce.trustAsHtml()` on user-provided content
- Whitelist template URLs for `ng-include`
- Do not loosen URL sanitization regexes without justification
- Sanitize data before passing to third-party DOM libraries

### Layer 4: Monitoring

- Enable CSP reporting to detect violations
- Log and alert on `$sanitize` stripping dangerous content
- Monitor for unusual template loading patterns

---

## Audit Checklist

Search the codebase for these patterns and verify each instance:

| Pattern | Risk | Action |
|---------|------|--------|
| `$sce.trustAsHtml` | High | Verify input is server-sanitized |
| `$sce.trustAsUrl` | High | Verify URL is from trusted source |
| `$sce.trustAsResourceUrl` | High | Verify URL is whitelisted |
| `$compile(` | Critical | Ensure no user content is compiled |
| `ng-bind-html-unsafe` | Critical | Remove — deprecated and unsafe |
| `$sceProvider.enabled(false)` | Critical | Remove — never disable SCE |
| `.html(` (jQuery) | High | Replace with `.text()` or `$sanitize` |
| `ng-include="` | Medium | Verify template URL is not user-controlled |
| `aHrefSanitizationTrustedUrlList` | Medium | Verify regex is not overly permissive |
