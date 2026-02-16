# Auth Patterns

## Protected Server Components

```tsx
// Direct auth check in Server Components — no middleware needed
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await auth();
  if (!session) redirect('/login');

  return <Dashboard user={session.user} />;
}

// Role-based access in Server Component
export default async function AdminPage() {
  const session = await auth();
  if (!session) redirect('/login');
  if (session.user.role !== 'admin') redirect('/unauthorized');

  return <AdminPanel />;
}

// Reusable auth wrapper
async function requireAuth(requiredRole?: string) {
  const session = await auth();
  if (!session) redirect('/login');
  if (requiredRole && session.user.role !== requiredRole) {
    redirect('/unauthorized');
  }
  return session;
}
```

## Protected Route Handler

```typescript
// app/api/users/route.ts
import { auth } from '@/lib/auth';

export async function GET() {
  const session = await auth();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  if (session.user.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  const users = await db.user.findMany({
    select: { id: true, name: true, email: true, role: true },
  });

  return NextResponse.json(users);
}
```

## Session Management

```typescript
// Server-side session access (Server Components, Route Handlers, Server Actions)
import { auth } from '@/lib/auth';

const session = await auth();
// session.user.id, session.user.email, session.user.role

// Client-side session access
'use client';
import { useSession } from 'next-auth/react';

export function UserMenu() {
  const { data: session, status } = useSession();

  if (status === 'loading') return <Skeleton />;
  if (!session) return <LoginButton />;

  return (
    <div>
      <span>{session.user.name}</span>
      <SignOutButton />
    </div>
  );
}

// Session provider (wrap in root layout)
// app/providers.tsx
'use client';
import { SessionProvider } from 'next-auth/react';

export function Providers({ children }: { children: React.ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>;
}
```

## OAuth Flow with Callback

```typescript
// Login page with multiple providers
// app/login/page.tsx
import { signIn } from '@/lib/auth';

export default function LoginPage({
  searchParams,
}: {
  searchParams: { callbackUrl?: string; error?: string };
}) {
  return (
    <div>
      {searchParams.error && (
        <div className="text-red-500">
          {searchParams.error === 'OAuthAccountNotLinked'
            ? 'Email already registered with different provider'
            : 'Authentication failed'}
        </div>
      )}

      <form action={async () => {
        'use server';
        await signIn('google', { redirectTo: searchParams.callbackUrl || '/dashboard' });
      }}>
        <button>Sign in with Google</button>
      </form>

      <form action={async (formData: FormData) => {
        'use server';
        await signIn('credentials', {
          email: formData.get('email'),
          password: formData.get('password'),
          redirectTo: searchParams.callbackUrl || '/dashboard',
        });
      }}>
        <input name="email" type="email" required />
        <input name="password" type="password" required />
        <button>Sign in</button>
      </form>
    </div>
  );
}
```

## JWT Best Practices

```typescript
// NextAuth.js JWT configuration
export const { handlers, auth } = NextAuth({
  session: { strategy: 'jwt' },
  jwt: {
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
  callbacks: {
    jwt({ token, user, trigger, session }) {
      // Initial sign in — add custom claims
      if (user) {
        token.role = user.role;
        token.permissions = user.permissions;
      }

      // Session update trigger (from update() call)
      if (trigger === 'update' && session) {
        token.name = session.name;
      }

      return token;
    },
    session({ session, token }) {
      // Only expose safe fields to client
      session.user.id = token.sub!;
      session.user.role = token.role as string;
      // Never expose: token.email, full permissions list, etc.
      return session;
    },
  },
});

// Security considerations:
// - JWT is stored in httpOnly cookie (default in NextAuth.js)
// - Token is encrypted with NEXTAUTH_SECRET
// - Use short maxAge for sensitive apps
// - Implement token rotation for long sessions
// - Never store sensitive data in JWT payload
```

## API Key Authentication (for webhooks)

```typescript
// app/api/webhooks/stripe/route.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get('stripe-signature')!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed':
      await handleCheckoutComplete(event.data.object);
      break;
    case 'invoice.payment_failed':
      await handlePaymentFailed(event.data.object);
      break;
  }

  return NextResponse.json({ received: true });
}
```
