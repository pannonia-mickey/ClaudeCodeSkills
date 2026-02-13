# Authentication Patterns for AngularJS

Comprehensive reference covering JWT handling, refresh token rotation, session management, OAuth integration, and secure storage patterns for AngularJS single-page applications.

---

## JWT Authentication Flow

### Token Storage

```javascript
// Use sessionStorage for session-lived tokens (cleared on tab close)
// Use localStorage only if persistent login is required

function TokenService($window) {
    var STORAGE = $window.sessionStorage;
    var TOKEN_KEY = 'access_token';
    var REFRESH_KEY = 'refresh_token';

    return {
        setTokens: function(access, refresh) {
            STORAGE.setItem(TOKEN_KEY, access);
            if (refresh) STORAGE.setItem(REFRESH_KEY, refresh);
        },
        getAccessToken: function() {
            return STORAGE.getItem(TOKEN_KEY);
        },
        getRefreshToken: function() {
            return STORAGE.getItem(REFRESH_KEY);
        },
        clearTokens: function() {
            STORAGE.removeItem(TOKEN_KEY);
            STORAGE.removeItem(REFRESH_KEY);
        },
        isTokenExpired: function(token) {
            if (!token) return true;
            try {
                var payload = JSON.parse(atob(token.split('.')[1]));
                return payload.exp * 1000 < Date.now();
            } catch (e) {
                return true;
            }
        }
    };
}
```

### Auth Interceptor with Token Refresh

```javascript
function AuthInterceptor($injector, $q, TokenService) {
    var isRefreshing = false;
    var requestQueue = [];

    return {
        request: function(config) {
            var token = TokenService.getAccessToken();
            if (token) {
                config.headers.Authorization = 'Bearer ' + token;
            }
            return config;
        },
        responseError: function(rejection) {
            if (rejection.status !== 401) {
                return $q.reject(rejection);
            }

            if (isRefreshing) {
                return queueRequest(rejection.config);
            }

            isRefreshing = true;
            var AuthService = $injector.get('AuthService');

            return AuthService.refreshToken()
                .then(function(newToken) {
                    rejection.config.headers.Authorization = 'Bearer ' + newToken;
                    retryQueuedRequests(newToken);
                    return $injector.get('$http')(rejection.config);
                })
                .catch(function() {
                    rejectQueuedRequests();
                    AuthService.logout();
                    $injector.get('$state').go('login');
                    return $q.reject(rejection);
                })
                .finally(function() {
                    isRefreshing = false;
                });
        }
    };

    function queueRequest(config) {
        var deferred = $q.defer();
        requestQueue.push({ config: config, deferred: deferred });
        return deferred.promise;
    }

    function retryQueuedRequests(token) {
        var $http = $injector.get('$http');
        requestQueue.forEach(function(item) {
            item.config.headers.Authorization = 'Bearer ' + token;
            item.deferred.resolve($http(item.config));
        });
        requestQueue = [];
    }

    function rejectQueuedRequests() {
        requestQueue.forEach(function(item) {
            item.deferred.reject('Session expired');
        });
        requestQueue = [];
    }
}
```

---

## Refresh Token Rotation

```javascript
function AuthService($http, $q, TokenService, API_URL) {
    return {
        login: function(credentials) {
            return $http.post(API_URL + '/auth/login', credentials)
                .then(function(response) {
                    TokenService.setTokens(
                        response.data.access_token,
                        response.data.refresh_token
                    );
                    return response.data.user;
                });
        },
        refreshToken: function() {
            var refreshToken = TokenService.getRefreshToken();
            if (!refreshToken) return $q.reject('No refresh token');

            return $http.post(API_URL + '/auth/refresh', {
                refresh_token: refreshToken
            }).then(function(response) {
                // Server issues new access AND refresh token (rotation)
                TokenService.setTokens(
                    response.data.access_token,
                    response.data.refresh_token
                );
                return response.data.access_token;
            });
        },
        logout: function() {
            var refreshToken = TokenService.getRefreshToken();
            TokenService.clearTokens();
            // Invalidate refresh token server-side
            if (refreshToken) {
                return $http.post(API_URL + '/auth/logout', {
                    refresh_token: refreshToken
                }).catch(function() {
                    // Best effort — tokens are already cleared locally
                });
            }
        }
    };
}
```

---

## OAuth 2.0 / OpenID Connect Integration

### Authorization Code Flow with PKCE

```javascript
function OAuthService($window, $httpParamSerializer, APP_CONFIG) {
    return {
        startLogin: function() {
            var codeVerifier = generateCodeVerifier();
            $window.sessionStorage.setItem('pkce_verifier', codeVerifier);

            return generateCodeChallenge(codeVerifier).then(function(codeChallenge) {
                var params = {
                    response_type: 'code',
                    client_id: APP_CONFIG.oauthClientId,
                    redirect_uri: APP_CONFIG.oauthRedirectUri,
                    scope: 'openid profile email',
                    state: generateRandomString(32),
                    code_challenge: codeChallenge,
                    code_challenge_method: 'S256'
                };

                $window.sessionStorage.setItem('oauth_state', params.state);
                $window.location.href = APP_CONFIG.oauthAuthorizeUrl + '?' +
                    $httpParamSerializer(params);
            });
        },
        handleCallback: function(code, state) {
            var storedState = $window.sessionStorage.getItem('oauth_state');
            if (state !== storedState) {
                throw new Error('OAuth state mismatch — possible CSRF attack');
            }

            var codeVerifier = $window.sessionStorage.getItem('pkce_verifier');
            $window.sessionStorage.removeItem('oauth_state');
            $window.sessionStorage.removeItem('pkce_verifier');

            return exchangeCode(code, codeVerifier);
        }
    };

    function generateCodeVerifier() {
        var array = new Uint8Array(32);
        $window.crypto.getRandomValues(array);
        return base64UrlEncode(array);
    }

    function generateCodeChallenge(verifier) {
        var encoder = new TextEncoder();
        var data = encoder.encode(verifier);
        return $window.crypto.subtle.digest('SHA-256', data)
            .then(function(hash) {
                return base64UrlEncode(new Uint8Array(hash));
            });
    }

    function generateRandomString(length) {
        var array = new Uint8Array(length);
        $window.crypto.getRandomValues(array);
        return base64UrlEncode(array).substring(0, length);
    }

    function base64UrlEncode(buffer) {
        return btoa(String.fromCharCode.apply(null, buffer))
            .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
    }
}
```

---

## Session Security Best Practices

### Idle Timeout

```javascript
function SessionTimeoutService($timeout, $transitions, AuthService, APP_CONFIG) {
    var timeoutPromise;
    var IDLE_TIMEOUT = APP_CONFIG.sessionIdleTimeout || 1800000; // 30 min

    return {
        start: start,
        reset: reset,
        stop: stop
    };

    function start() {
        // Reset timer on every state transition
        $transitions.onSuccess({}, function() {
            reset();
        });
        reset();
    }

    function reset() {
        stop();
        timeoutPromise = $timeout(function() {
            AuthService.logout();
        }, IDLE_TIMEOUT);
    }

    function stop() {
        if (timeoutPromise) {
            $timeout.cancel(timeoutPromise);
        }
    }
}
```

### Cross-Tab Session Sync

```javascript
// Sync logout across browser tabs via storage events
angular.module('app.core')
    .run(['$window', 'AuthService', '$state', function($window, AuthService, $state) {
        $window.addEventListener('storage', function(event) {
            if (event.key === 'auth_token' && event.newValue === null) {
                AuthService.logout();
                $state.go('login');
            }
        });
    }]);
```

---

## Security Headers Checklist

Ensure the server sends these headers for the AngularJS SPA:

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `X-XSS-Protection` | `0` | Disable browser XSS filter (use CSP instead) |
| `Content-Security-Policy` | See CSP section | Prevent XSS, injection |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limit referrer leaks |
| `Cache-Control` | `no-store` (for API responses) | Prevent sensitive data caching |
