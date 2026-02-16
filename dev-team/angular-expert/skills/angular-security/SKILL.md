---
name: Angular Security
description: This skill should be used when the user asks to "secure Angular app", "Angular XSS prevention", "Angular CSRF", "Angular authentication", "Angular route guards", "Angular CSP", "DomSanitizer", or "Angular security headers". It covers XSS prevention, CSRF, authentication patterns, route protection, and CSP.
---

# Angular Security

## XSS Prevention

Angular automatically sanitizes values bound in templates. The framework escapes HTML, attributes, styles, and URLs by default.

```typescript
// SAFE — Angular auto-escapes
<p>{{ userInput }}</p>
<div [innerHTML]="userContent"></div>  <!-- Sanitized by Angular -->

// DANGER — bypass sanitization only with trusted content
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Component({ /* ... */ })
export class RichContentComponent {
  private sanitizer = inject(DomSanitizer);

  // Only use for content you control (e.g., CMS with server-side sanitization)
  trustedHtml: SafeHtml;
  constructor() {
    this.trustedHtml = this.sanitizer.bypassSecurityTrustHtml(serverSanitizedHtml);
  }
}
```

**Rules:**
- Never use `bypassSecurityTrust*` with user-provided content
- Use `[innerHTML]` binding instead of `bypassSecurityTrustHtml` when possible (Angular sanitizes it)
- Avoid `document.createElement` and direct DOM manipulation

## CSRF/XSRF Protection

Angular's HttpClient has built-in XSRF support:

```typescript
// app.config.ts — configure XSRF cookie/header names
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',     // Cookie set by server
        headerName: 'X-XSRF-TOKEN',   // Header sent by Angular
      })
    ),
  ],
};
```

The server must set a `XSRF-TOKEN` cookie; Angular reads it and sends it as the `X-XSRF-TOKEN` header on mutating requests (POST, PUT, DELETE).

## Authentication Pattern (JWT)

### Auth Service

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);
  private router = inject(Router);

  private tokenKey = 'access_token';
  isAuthenticated = signal(!!localStorage.getItem(this.tokenKey));
  currentUser = signal<User | null>(null);

  login(credentials: LoginRequest): Observable<TokenResponse> {
    return this.http.post<TokenResponse>('/api/auth/login', credentials).pipe(
      tap(response => {
        localStorage.setItem(this.tokenKey, response.accessToken);
        this.isAuthenticated.set(true);
        this.currentUser.set(response.user);
      })
    );
  }

  logout() {
    localStorage.removeItem(this.tokenKey);
    this.isAuthenticated.set(false);
    this.currentUser.set(null);
    this.router.navigate(['/login']);
  }

  getToken(): string | null {
    return localStorage.getItem(this.tokenKey);
  }
}
```

### Auth Interceptor

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }

  return next(req).pipe(
    catchError(error => {
      if (error.status === 401) {
        authService.logout();
      }
      return throwError(() => error);
    })
  );
};
```

### Auth Guard

```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) return true;
  return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const requiredRoles = route.data['roles'] as string[];
  const user = authService.currentUser();
  return user !== null && requiredRoles.some(role => user.roles.includes(role));
};

// Usage in routes
{ path: 'admin', component: AdminComponent, canActivate: [authGuard, roleGuard], data: { roles: ['admin'] } }
```

## Content Security Policy

```html
<!-- index.html — use nonce for inline scripts -->
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' 'nonce-{{CSP_NONCE}}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
">
```

For Angular CLI builds, configure CSP in the server response headers rather than meta tags for production.

## Secure HTTP Communication

```typescript
// Always use HTTPS
export const environment = {
  apiUrl: 'https://api.example.com', // Never http:// in production
};

// Validate URLs before navigation
const isSafeUrl = (url: string) => url.startsWith('/') && !url.startsWith('//');
```

## Dependency Auditing

```bash
# Check for known vulnerabilities
npm audit
ng update  # Keep Angular packages up to date

# Use strict mode for new projects
ng new my-app --strict
```

## References

- [Auth Patterns](references/auth-patterns.md) — Complete JWT flow, OAuth2/OIDC, refresh tokens, session management.
- [Security Hardening](references/security-hardening.md) — CSP nonce strategy, trusted types, subresource integrity, security headers.
