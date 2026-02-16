# Angular Security Hardening

## Content Security Policy (CSP)

### Server-Side Headers (Recommended)

```nginx
# nginx.conf
add_header Content-Security-Policy "
  default-src 'self';
  script-src 'self' 'nonce-$request_id';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com wss://ws.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests;
" always;
```

### Angular CLI Nonce Support

```typescript
// main.ts — CSP nonce for Angular styles
const nonce = document.querySelector('meta[name="csp-nonce"]')?.getAttribute('content');
if (nonce) {
  bootstrapApplication(AppComponent, {
    ...appConfig,
    providers: [...appConfig.providers, { provide: CSP_NONCE, useValue: nonce }],
  });
}
```

```html
<!-- index.html -->
<meta name="csp-nonce" content="{{CSP_NONCE}}">
```

## Trusted Types

```typescript
// app.config.ts — enforce trusted types policy
if (window.trustedTypes) {
  window.trustedTypes.createPolicy('angular', {
    createHTML: (input: string) => input, // Angular handles sanitization
    createScript: () => { throw new Error('Script creation not allowed'); },
    createScriptURL: () => { throw new Error('Script URL creation not allowed'); },
  });
}
```

```
# CSP header with trusted types
Content-Security-Policy: trusted-types angular; require-trusted-types-for 'script';
```

## Subresource Integrity (SRI)

```json
// angular.json — enable SRI for production builds
{
  "configurations": {
    "production": {
      "subresourceIntegrity": true,
      "outputHashing": "all"
    }
  }
}
```

This generates:
```html
<script src="main.abc123.js" integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8w" crossorigin="anonymous"></script>
```

## Security Headers

```nginx
# Complete security headers for Angular app
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header X-XSS-Protection "0" always;  # Disabled; CSP replaces it
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(self)" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Embedder-Policy "require-corp" always;
add_header Cross-Origin-Resource-Policy "same-origin" always;
```

## Sanitization Best Practices

```typescript
// Angular auto-sanitizes these bindings:
// [innerHTML] — sanitizes HTML, removes scripts
// [href] — sanitizes URLs, blocks javascript:
// [src] — sanitizes URLs
// [style] — sanitizes CSS

// NEVER bypass sanitization with user input
// BAD:
this.sanitizer.bypassSecurityTrustHtml(userInput); // XSS vulnerability!

// GOOD: Only bypass for content you fully control
@Component({
  template: `<div [innerHTML]="safeContent"></div>`,
})
export class SafeComponent {
  private sanitizer = inject(DomSanitizer);

  // Only for server-sanitized CMS content
  safeContent = this.sanitizer.bypassSecurityTrustHtml(this.serverSanitizedContent);
}

// BEST: Let Angular handle sanitization
@Component({
  template: `<div [innerHTML]="userContent"></div>`, // Angular sanitizes automatically
})
export class BetterComponent {
  userContent = '<b>Bold</b><script>alert("xss")</script>'; // Script stripped by Angular
}
```

## Secure HTTP Communication

```typescript
// HTTPS-only API configuration
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com', // Always HTTPS
};

// Force HTTPS redirect (server-side)
// nginx: return 301 https://$host$request_uri;

// Cookie security for session tokens
// Server should set: Set-Cookie: token=xxx; Secure; HttpOnly; SameSite=Strict; Path=/

// CORS configuration (server-side)
// Access-Control-Allow-Origin: https://app.example.com
// Access-Control-Allow-Methods: GET, POST, PUT, DELETE
// Access-Control-Allow-Headers: Authorization, Content-Type
// Access-Control-Allow-Credentials: true
```

## Safe URL Handling

```typescript
// Validate redirect URLs to prevent open redirect
export function isSafeRedirectUrl(url: string): boolean {
  // Only allow relative URLs starting with /
  if (!url.startsWith('/')) return false;
  // Block protocol-relative URLs
  if (url.startsWith('//')) return false;
  // Block encoded characters that could bypass checks
  try {
    const decoded = decodeURIComponent(url);
    if (decoded.startsWith('//') || decoded.includes('://')) return false;
  } catch {
    return false;
  }
  return true;
}

// Usage in login redirect
@Component({ /* ... */ })
export class LoginComponent {
  private route = inject(ActivatedRoute);
  private router = inject(Router);

  onLoginSuccess() {
    const returnUrl = this.route.snapshot.queryParams['returnUrl'] ?? '/dashboard';
    const safeUrl = isSafeRedirectUrl(returnUrl) ? returnUrl : '/dashboard';
    this.router.navigateByUrl(safeUrl);
  }
}
```

## Input Validation

```typescript
// Server-side validation is always required, but client-side improves UX
export class SecureFormComponent {
  form = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    name: new FormControl('', [
      Validators.required,
      Validators.maxLength(100),
      Validators.pattern(/^[a-zA-Z\s\-']+$/), // Whitelist characters
    ]),
    url: new FormControl('', [
      Validators.pattern(/^https:\/\/.+/), // Enforce HTTPS
    ]),
  });
}

// Sanitize data before sending to API
export function sanitizeInput(input: string): string {
  return input.trim().replace(/\s+/g, ' ');
}
```

## Dependency Security

```bash
# Regular security audits
npm audit
npm audit fix

# Check for outdated Angular packages
ng update

# Use lockfile for deterministic builds
npm ci  # Always use in CI

# Pin dependency versions
# package.json — use exact versions for critical deps
"dependencies": {
  "@angular/core": "18.2.0",  # Not ^18.2.0
}
```

## Angular-Specific Security Checklist

- [ ] All components use `ChangeDetectionStrategy.OnPush`
- [ ] No `bypassSecurityTrust*` calls with user input
- [ ] No direct DOM manipulation (`document.createElement`, `innerHTML` assignment)
- [ ] All route guards in place for protected routes
- [ ] XSRF configuration enabled for API calls
- [ ] CSP headers configured on server
- [ ] SRI enabled for production builds
- [ ] HTTPS enforced for all API endpoints
- [ ] Redirect URLs validated before navigation
- [ ] Input validation on all forms
- [ ] Regular `npm audit` in CI pipeline
- [ ] Angular packages kept up to date
