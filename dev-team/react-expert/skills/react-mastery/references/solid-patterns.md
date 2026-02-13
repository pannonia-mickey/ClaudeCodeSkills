# SOLID Principles for React with TypeScript

## Single Responsibility Principle (SRP)

### Single-Purpose Components

Every component should have one reason to change. When a component handles both data fetching and rendering, extract the data logic into a custom hook or move it to a Server Component.

```typescript
// BAD: Component does too many things
function UserDashboard() {
  const [user, setUser] = useState<User | null>(null);
  const [orders, setOrders] = useState<Order[]>([]);
  const [notifications, setNotifications] = useState<Notification[]>([]);

  useEffect(() => { /* fetch user */ }, []);
  useEffect(() => { /* fetch orders */ }, []);
  useEffect(() => { /* fetch notifications */ }, []);

  return (
    <div>
      {/* renders user info, order table, notification list, sidebar, etc. */}
    </div>
  );
}

// GOOD: Each component has a single responsibility
function UserDashboard() {
  return (
    <DashboardLayout>
      <UserProfile />
      <OrderHistory />
      <NotificationFeed />
    </DashboardLayout>
  );
}

function UserProfile() {
  const { data: user } = useUser();
  return <ProfileCard user={user} />;
}
```

### Custom Hooks for Logic Extraction

Extract stateful logic into hooks that each serve a single purpose.

```typescript
function useFormField(initialValue: string, validate: (value: string) => string | null) {
  const [value, setValue] = useState(initialValue);
  const [error, setError] = useState<string | null>(null);
  const [touched, setTouched] = useState(false);

  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
    if (touched) {
      setError(validate(e.target.value));
    }
  }, [touched, validate]);

  const handleBlur = useCallback(() => {
    setTouched(true);
    setError(validate(value));
  }, [value, validate]);

  return { value, error, touched, handleChange, handleBlur };
}
```

Each hook manages one concern. Compose hooks in components rather than building one hook that manages all form state, validation, submission, and error handling.

## Open/Closed Principle (OCP)

### Composition Over Inheritance

Components should be open for extension through composition but closed for modification. Use children, render props, and slots to allow extension without changing the component source.

```typescript
// GOOD: Open for extension via composition
interface CardProps {
  children: React.ReactNode;
  className?: string;
}

function Card({ children, className }: CardProps) {
  return <div className={clsx(styles.card, className)}>{children}</div>;
}

// Extend via composition, not modification
function UserCard({ user }: { user: User }) {
  return (
    <Card>
      <Card.Header>
        <Avatar src={user.avatar} />
        <h3>{user.name}</h3>
      </Card.Header>
      <Card.Body>
        <p>{user.bio}</p>
      </Card.Body>
    </Card>
  );
}
```

### Higher-Order Components (HOCs)

Use HOCs to add cross-cutting concerns without modifying the original component.

```typescript
function withErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  fallback: React.ReactNode
) {
  return function WithErrorBoundary(props: P) {
    return (
      <ErrorBoundary fallback={fallback}>
        <WrappedComponent {...props} />
      </ErrorBoundary>
    );
  };
}

const SafeDashboard = withErrorBoundary(Dashboard, <DashboardError />);
```

### The Children Pattern

Accept `children` to let consumers control content without the parent needing to know what it renders.

```typescript
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
  footer?: React.ReactNode;
}

function Modal({ isOpen, onClose, title, children, footer }: ModalProps) {
  if (!isOpen) return null;
  return (
    <dialog open aria-labelledby="modal-title">
      <h2 id="modal-title">{title}</h2>
      <div className={styles.body}>{children}</div>
      {footer && <div className={styles.footer}>{footer}</div>}
    </dialog>
  );
}
```

## Liskov Substitution Principle (LSP)

### Props Contracts with TypeScript Interfaces

Any component that implements a given props interface must honor the full contract. Subtypes should be usable wherever the base type is expected.

```typescript
interface InputProps {
  value: string;
  onChange: (value: string) => void;
  error?: string;
  disabled?: boolean;
  placeholder?: string;
}

// Both TextInput and TextArea satisfy InputProps
function TextInput({ value, onChange, error, disabled, placeholder }: InputProps) {
  return (
    <div>
      <input
        value={value}
        onChange={(e) => onChange(e.target.value)}
        disabled={disabled}
        placeholder={placeholder}
        aria-invalid={!!error}
      />
      {error && <span role="alert">{error}</span>}
    </div>
  );
}

function TextArea({ value, onChange, error, disabled, placeholder }: InputProps) {
  return (
    <div>
      <textarea
        value={value}
        onChange={(e) => onChange(e.target.value)}
        disabled={disabled}
        placeholder={placeholder}
        aria-invalid={!!error}
      />
      {error && <span role="alert">{error}</span>}
    </div>
  );
}
```

### Consistent APIs and forwardRef

When wrapping native elements, forward refs and spread remaining props to maintain substitutability with the native element.

```typescript
interface CustomInputProps extends Omit<React.InputHTMLAttributes<HTMLInputElement>, 'onChange'> {
  label: string;
  error?: string;
  onChange: (value: string) => void;
}

const CustomInput = React.forwardRef<HTMLInputElement, CustomInputProps>(
  function CustomInput({ label, error, onChange, className, ...rest }, ref) {
    const id = useId();
    return (
      <div className={className}>
        <label htmlFor={id}>{label}</label>
        <input
          ref={ref}
          id={id}
          onChange={(e) => onChange(e.target.value)}
          aria-describedby={error ? `${id}-error` : undefined}
          aria-invalid={!!error}
          {...rest}
        />
        {error && <span id={`${id}-error`} role="alert">{error}</span>}
      </div>
    );
  }
);
```

## Interface Segregation Principle (ISP)

### Granular Context Providers

Do not put all application state into one massive context. Split contexts by domain so consumers only subscribe to the data they need.

```typescript
// BAD: One giant context
const AppContext = createContext<{
  user: User; theme: Theme; locale: string; notifications: Notification[];
  /* ...20 more fields */
} | null>(null);

// GOOD: Granular, focused contexts
const AuthContext = createContext<AuthContextValue | null>(null);
const ThemeContext = createContext<ThemeContextValue | null>(null);
const LocaleContext = createContext<LocaleContextValue | null>(null);

function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

When a context value changes, every consumer re-renders. Splitting contexts means a theme change does not re-render components that only use auth state.

### Small Focused Hooks

Build hooks that return the minimum necessary interface. Consumers should not depend on data they do not use.

```typescript
// BAD: Returns everything
function useUser() {
  // returns { user, orders, preferences, notifications, updateUser, deleteUser, ... }
}

// GOOD: Focused hooks
function useUserProfile() {
  // returns { name, avatar, bio }
}

function useUserOrders() {
  // returns { orders, isLoading, refetch }
}

function useUserPreferences() {
  // returns { theme, locale, updatePreferences }
}
```

## Dependency Inversion Principle (DIP)

### Context-Based Dependency Injection

High-level components should not depend on concrete implementations. Use context to inject dependencies.

```typescript
interface AnalyticsService {
  track(event: string, properties?: Record<string, unknown>): void;
  identify(userId: string, traits?: Record<string, unknown>): void;
}

const AnalyticsContext = createContext<AnalyticsService | null>(null);

function useAnalytics(): AnalyticsService {
  const context = useContext(AnalyticsContext);
  if (!context) throw new Error('useAnalytics must be used within AnalyticsProvider');
  return context;
}

// Production provider
function AnalyticsProvider({ children }: { children: React.ReactNode }) {
  const service = useMemo<AnalyticsService>(() => ({
    track: (event, props) => mixpanel.track(event, props),
    identify: (userId, traits) => mixpanel.identify(userId, traits),
  }), []);

  return <AnalyticsContext.Provider value={service}>{children}</AnalyticsContext.Provider>;
}

// Test provider
function MockAnalyticsProvider({ children }: { children: React.ReactNode }) {
  const service = useMemo<AnalyticsService>(() => ({
    track: vi.fn(),
    identify: vi.fn(),
  }), []);

  return <AnalyticsContext.Provider value={service}>{children}</AnalyticsContext.Provider>;
}
```

### The Provider Pattern

Compose providers to build an application shell where high-level features depend on abstractions, not concretions.

```typescript
function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary fallback={<CriticalError />}>
      <QueryClientProvider client={queryClient}>
        <AuthProvider>
          <ThemeProvider>
            <AnalyticsProvider>
              <ToastProvider>
                {children}
              </ToastProvider>
            </AnalyticsProvider>
          </ThemeProvider>
        </AuthProvider>
      </QueryClientProvider>
    </ErrorBoundary>
  );
}
```

### Hook-Based Abstractions

Abstract external dependencies behind custom hooks so components depend on a stable interface, not a specific library.

```typescript
// Abstract the data-fetching library behind a hook
function useUserData(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
  });
}

// Components depend on useUserData, not on TanStack Query directly
// If you swap to SWR or another library, only the hook implementation changes
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useUserData(userId);

  if (isLoading) return <ProfileSkeleton />;
  if (error) return <ProfileError error={error} />;
  return <ProfileCard user={user} />;
}
```

This means migrating from one data-fetching library to another requires changing only the hook internals, not every component that uses the data.
