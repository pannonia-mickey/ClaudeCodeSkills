# Angular Authentication Patterns

## Complete JWT Flow

```typescript
// models
interface TokenResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  user: User;
}

interface User {
  id: string;
  email: string;
  roles: string[];
}

// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);
  private router = inject(Router);

  private tokenKey = 'access_token';
  private refreshKey = 'refresh_token';

  isAuthenticated = signal(!!this.getToken());
  currentUser = signal<User | null>(this.getUserFromToken());

  login(credentials: { email: string; password: string }): Observable<TokenResponse> {
    return this.http.post<TokenResponse>('/api/auth/login', credentials).pipe(
      tap(response => this.setSession(response))
    );
  }

  logout() {
    this.http.post('/api/auth/logout', { refreshToken: this.getRefreshToken() }).subscribe();
    this.clearSession();
    this.router.navigate(['/login']);
  }

  refreshAccessToken(): Observable<TokenResponse> {
    const refreshToken = this.getRefreshToken();
    if (!refreshToken) {
      this.clearSession();
      return throwError(() => new Error('No refresh token'));
    }

    return this.http.post<TokenResponse>('/api/auth/refresh', { refreshToken }).pipe(
      tap(response => this.setSession(response)),
      catchError(error => {
        this.clearSession();
        return throwError(() => error);
      })
    );
  }

  getToken(): string | null {
    return localStorage.getItem(this.tokenKey);
  }

  isTokenExpired(): boolean {
    const token = this.getToken();
    if (!token) return true;
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.exp * 1000 < Date.now();
  }

  hasRole(role: string): boolean {
    return this.currentUser()?.roles.includes(role) ?? false;
  }

  private setSession(response: TokenResponse) {
    localStorage.setItem(this.tokenKey, response.accessToken);
    localStorage.setItem(this.refreshKey, response.refreshToken);
    this.isAuthenticated.set(true);
    this.currentUser.set(response.user);
  }

  private clearSession() {
    localStorage.removeItem(this.tokenKey);
    localStorage.removeItem(this.refreshKey);
    this.isAuthenticated.set(false);
    this.currentUser.set(null);
  }

  private getRefreshToken(): string | null {
    return localStorage.getItem(this.refreshKey);
  }

  private getUserFromToken(): User | null {
    const token = this.getToken();
    if (!token) return null;
    try {
      const payload = JSON.parse(atob(token.split('.')[1]));
      return { id: payload.sub, email: payload.email, roles: payload.roles ?? [] };
    } catch {
      return null;
    }
  }
}
```

## Refresh Token Interceptor

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);

  // Skip auth endpoints
  if (req.url.includes('/auth/login') || req.url.includes('/auth/refresh')) {
    return next(req);
  }

  const token = authService.getToken();
  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }

  return next(req).pipe(
    catchError(error => {
      if (error.status === 401 && !req.url.includes('/auth/refresh')) {
        return authService.refreshAccessToken().pipe(
          switchMap(response => {
            const retryReq = req.clone({
              setHeaders: { Authorization: `Bearer ${response.accessToken}` },
            });
            return next(retryReq);
          }),
          catchError(refreshError => {
            authService.logout();
            return throwError(() => refreshError);
          })
        );
      }
      return throwError(() => error);
    })
  );
};
```

## OAuth2/OIDC with angular-auth-oidc-client

```typescript
// app.config.ts
import { provideAuth, withDefaultFeatures } from 'angular-auth-oidc-client';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAuth({
      config: {
        authority: 'https://auth.example.com',
        redirectUrl: window.location.origin,
        postLogoutRedirectUri: window.location.origin,
        clientId: 'my-angular-app',
        scope: 'openid profile email',
        responseType: 'code',
        silentRenew: true,
        useRefreshToken: true,
        renewTimeBeforeTokenExpiresInSec: 30,
      },
    }),
  ],
};

// auth-oidc.service.ts
@Injectable({ providedIn: 'root' })
export class AuthOidcService {
  private oidcService = inject(OidcSecurityService);

  isAuthenticated$ = this.oidcService.isAuthenticated$;
  userData$ = this.oidcService.userData$;
  token$ = this.oidcService.getAccessToken();

  login() { this.oidcService.authorize(); }
  logout() { this.oidcService.logoff().subscribe(); }

  checkAuth(): Observable<LoginResponse> {
    return this.oidcService.checkAuth();
  }
}

// app.component.ts
export class AppComponent implements OnInit {
  private authService = inject(AuthOidcService);

  ngOnInit() {
    this.authService.checkAuth().subscribe();
  }
}
```

## Route Guards

```typescript
// Functional auth guard
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) return true;
  return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

// Role-based guard
export const roleGuard: CanActivateFn = (route) => {
  const auth = inject(AuthService);
  const requiredRoles: string[] = route.data['roles'] ?? [];
  return requiredRoles.length === 0 || requiredRoles.some(role => auth.hasRole(role));
};

// Permission-based guard
export const permissionGuard: CanActivateFn = (route) => {
  const auth = inject(AuthService);
  const requiredPermission: string = route.data['permission'];
  return auth.currentUser()?.permissions?.includes(requiredPermission) ?? false;
};

// canMatch — prevents lazy-loaded chunk from downloading
export const adminMatch: CanMatchFn = () => {
  return inject(AuthService).hasRole('admin');
};

// Route configuration
export const routes: Routes = [
  { path: 'dashboard', component: DashboardComponent, canActivate: [authGuard] },
  { path: 'admin', loadChildren: () => import('./admin/admin.routes'), canMatch: [adminMatch], canActivate: [authGuard, roleGuard], data: { roles: ['admin'] } },
  { path: 'reports', component: ReportsComponent, canActivate: [authGuard, permissionGuard], data: { permission: 'reports:view' } },
];
```

## Session Management

```typescript
@Injectable({ providedIn: 'root' })
export class SessionService {
  private auth = inject(AuthService);
  private destroyRef = inject(DestroyRef);

  private idleTimeout = 15 * 60 * 1000; // 15 minutes
  private idleTimer?: ReturnType<typeof setTimeout>;

  startIdleTracking() {
    const events = ['mousedown', 'keydown', 'scroll', 'touchstart'];

    merge(...events.map(e => fromEvent(document, e))).pipe(
      throttleTime(1000),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(() => this.resetIdleTimer());

    this.resetIdleTimer();
  }

  private resetIdleTimer() {
    if (this.idleTimer) clearTimeout(this.idleTimer);
    this.idleTimer = setTimeout(() => {
      this.auth.logout();
    }, this.idleTimeout);
  }
}
```
