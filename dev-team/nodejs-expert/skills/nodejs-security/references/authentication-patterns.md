# Node.js Authentication Patterns

## Complete JWT Flow (Access + Refresh)

```typescript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';

@Injectable()
class AuthService {
  constructor(
    private userRepo: UserRepository,
    private tokenRepo: RefreshTokenRepository,
  ) {}

  async login(email: string, password: string) {
    const user = await this.userRepo.findByEmail(email);
    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      throw new UnauthorizedError('Invalid credentials');
    }

    const { accessToken, refreshToken } = this.generateTokens(user);

    // Store refresh token hash in DB (enables revocation)
    const tokenHash = await bcrypt.hash(refreshToken, 10);
    await this.tokenRepo.create({ userId: user.id, tokenHash, expiresAt: addDays(7) });

    return { accessToken, refreshToken, user: this.sanitizeUser(user) };
  }

  async refresh(refreshToken: string) {
    const payload = jwt.verify(refreshToken, config.jwt.refreshSecret) as { sub: string };

    const stored = await this.tokenRepo.findByUserId(payload.sub);
    if (!stored || !(await bcrypt.compare(refreshToken, stored.tokenHash))) {
      throw new UnauthorizedError('Invalid refresh token');
    }

    // Rotate refresh token
    const user = await this.userRepo.findById(payload.sub);
    const tokens = this.generateTokens(user!);

    await this.tokenRepo.update(stored.id, {
      tokenHash: await bcrypt.hash(tokens.refreshToken, 10),
      expiresAt: addDays(7),
    });

    return tokens;
  }

  async logout(userId: string) {
    await this.tokenRepo.deleteByUserId(userId);
  }

  private generateTokens(user: User) {
    return {
      accessToken: jwt.sign({ sub: user.id, role: user.role }, config.jwt.secret, { expiresIn: '15m' }),
      refreshToken: jwt.sign({ sub: user.id }, config.jwt.refreshSecret, { expiresIn: '7d' }),
    };
  }
}
```

## Passport.js Strategies

```typescript
import passport from 'passport';
import { Strategy as JwtStrategy, ExtractJwt } from 'passport-jwt';
import { Strategy as LocalStrategy } from 'passport-local';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';

// Local (email/password)
passport.use(new LocalStrategy(
  { usernameField: 'email' },
  async (email, password, done) => {
    const user = await userRepo.findByEmail(email);
    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      return done(null, false, { message: 'Invalid credentials' });
    }
    done(null, user);
  },
));

// JWT
passport.use(new JwtStrategy(
  {
    jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
    secretOrKey: config.jwt.secret,
  },
  async (payload, done) => {
    const user = await userRepo.findById(payload.sub);
    done(null, user || false);
  },
));

// Google OAuth
passport.use(new GoogleStrategy(
  {
    clientID: config.google.clientId,
    clientSecret: config.google.clientSecret,
    callbackURL: '/api/auth/google/callback',
  },
  async (accessToken, refreshToken, profile, done) => {
    let user = await userRepo.findByGoogleId(profile.id);
    if (!user) {
      user = await userRepo.create({
        email: profile.emails![0].value,
        name: profile.displayName,
        googleId: profile.id,
      });
    }
    done(null, user);
  },
));
```

## RBAC Implementation

```typescript
type Role = 'admin' | 'editor' | 'viewer';
type Permission = 'users:read' | 'users:write' | 'posts:read' | 'posts:write' | 'posts:delete';

const rolePermissions: Record<Role, Permission[]> = {
  admin: ['users:read', 'users:write', 'posts:read', 'posts:write', 'posts:delete'],
  editor: ['posts:read', 'posts:write'],
  viewer: ['posts:read'],
};

function requirePermission(...permissions: Permission[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user?.role as Role;
    const userPermissions = rolePermissions[userRole] ?? [];
    const hasAll = permissions.every(p => userPermissions.includes(p));
    if (!hasAll) return res.status(403).json({ error: 'Insufficient permissions' });
    next();
  };
}

// Usage
router.delete('/posts/:id', authenticate, requirePermission('posts:delete'), deletePost);
```

## API Key Authentication

```typescript
function apiKeyAuth(req: Request, res: Response, next: NextFunction) {
  const apiKey = req.headers['x-api-key'] as string;
  if (!apiKey) return res.status(401).json({ error: 'API key required' });

  // Compare using timing-safe comparison to prevent timing attacks
  const validKey = Buffer.from(config.apiKey);
  const providedKey = Buffer.from(apiKey);

  if (validKey.length !== providedKey.length || !crypto.timingSafeEqual(validKey, providedKey)) {
    return res.status(401).json({ error: 'Invalid API key' });
  }

  next();
}
```
