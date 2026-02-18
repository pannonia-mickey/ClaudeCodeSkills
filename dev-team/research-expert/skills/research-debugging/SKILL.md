---
name: Research Debugging
triggers:
  - "getting an error"
  - "how to fix"
  - "TypeError"
  - "why is this failing"
  - "cannot read"
  - "undefined is not"
  - "unexpected token"
  - "module not found"
  - "permission denied"
  - "debugging"
description: |
  This skill should be used when the research-expert needs to investigate an error message, exception, or unexpected behavior — decomposing the error into its root cause, finding verified solutions from authoritative sources, ranking them by reliability, and presenting actionable fixes with code examples and preventive patterns.
---

# Research Debugging: Error Research and Solution Synthesis

## Error Decomposition

Before searching for solutions, decompose the error into its essential components. This determines the best search strategy.

### Decomposition Framework

```
Error: "TypeError: Cannot read properties of undefined (reading 'map')"

1. Error type:    TypeError — a JavaScript runtime type error
2. Operation:     Reading property 'map' — accessing an array method
3. Subject:       undefined — the value being accessed is undefined
4. Context clues: After a state update in React (from user description)

→ Root cause category: Uninitialized state / async data not yet loaded
→ Search queries:
    "Cannot read properties of undefined reading map react"
    "react state undefined map TypeError"
    "react useEffect async undefined state"
```

### Error Type → Root Cause Categories

| Error Type | Common Root Causes |
|-----------|-------------------|
| `TypeError: X is not a function` | Wrong import, using before initialization, API changed |
| `TypeError: Cannot read properties of undefined` | Async data not loaded, optional chaining missing, wrong property name |
| `ReferenceError: X is not defined` | Missing import, typo, wrong scope, build issue |
| `SyntaxError: Unexpected token` | JSON parse error, wrong file extension, babel config issue |
| `ENOENT: no such file or directory` | Wrong path, missing file, working directory wrong |
| `EACCES: permission denied` | File system permissions, running as wrong user |
| `ECONNREFUSED` | Service not running, wrong port, firewall |
| `ETIMEDOUT` | Network issue, service overloaded, firewall blocking |
| `ModuleNotFoundError` (Python) | Missing dependency, wrong virtual env, wrong import path |
| `ImportError` (Python) | Wrong module name, circular import, missing `__init__.py` |
| `404 Not Found` | Wrong URL, route not registered, base path wrong |
| `401 Unauthorized` | Missing/invalid token, token expired |
| `403 Forbidden` | Valid auth but insufficient permissions |
| `500 Internal Server Error` | Unhandled exception server-side — check server logs |

---

## Targeted Search Queries

### Query Construction Rules

**Rule 1 — Use the exact error message in quotes:**
```
"Cannot read properties of undefined (reading 'map')"
```
This finds people with the exact same error, not related ones.

**Rule 2 — Add library + version:**
```
"Cannot read properties of undefined" react 18 useEffect
```
Distinguishes version-specific causes.

**Rule 3 — Add what you were doing:**
```
"Cannot read properties of undefined" react fetch async state
```
Narrows to your specific scenario.

**Rule 4 — For library errors, search GitHub Issues:**
```
site:github.com/{owner}/{repo}/issues "exact error message"
```
Maintainer-answered issues are often more specific than SO.

**Rule 5 — For version-specific bugs, search changelog/releases:**
```
{library} github releases "fixed" {short error description}
```
May reveal a known bug with a version upgrade as the fix.

### Query Templates by Error Type

```bash
# JavaScript runtime errors
"TypeError: {exact message}" {framework/library} {version}
"TypeError: {exact message}" site:stackoverflow.com

# Module/import errors
"Cannot find module '{module-name}'" {bundler/runtime}
"{module} not found" {package-manager} fix

# Network/HTTP errors
"{library} {HTTP method} {status-code} error handling"
"fetch {status-code} {library} retry"

# Database errors
"{ORM} {error message}" site:github.com/{orm-repo}/issues
"{ORM} {error type} {operation}" solution

# Build/compilation errors
"{compiler/bundler} error: {message}" fix
"{error code}" {build tool} {version}
```

---

## Solution Ranking

Not all solutions are equal. Rank them before presenting.

### Reliability Hierarchy

1. **Official docs fix** — Documentation explicitly says "if you see X, do Y"
   - Signal: Official docs page with a troubleshooting section
   - Confidence: Highest

2. **Maintainer response in GitHub Issues** — Core team member explains and provides fix
   - Signal: Comment from contributor with "member" or "owner" badge
   - Confidence: Very High

3. **Accepted Stack Overflow answer** — Accepted + high votes
   - Signal: Green checkmark + 50+ votes + recent (< 2 years for fast ecosystems)
   - Confidence: High

4. **Multiple independent sources agreeing** — 3+ sources suggest the same fix
   - Signal: Different authors, different platforms, same approach
   - Confidence: High

5. **High-vote but not accepted SO answer** — Community consensus
   - Signal: 20+ votes, comments confirm it works
   - Confidence: Medium-High

6. **Single community answer** — One blog post or comment
   - Signal: Author has credibility in the domain
   - Confidence: Medium

7. **Workaround from issues/discussions** — Community-found hack
   - Signal: Upvoted comment in GitHub issue
   - Confidence: Medium (but note it as a workaround, not a fix)

### When Solutions Conflict

If Rank-3 and Rank-5 solutions contradict each other:
1. Check if they target different versions (most conflicts are version differences)
2. Try both mentally — does one obviously handle edge cases better?
3. Present both with explicit version requirements

---

## Solution Presentation Format

Structure debug solutions for immediate action.

### Standard Debug Response

```markdown
## Error: {Short Error Description}

**Root Cause**: {One sentence — what actually causes this error}

**Most Common Fix** (applies in ~{X}% of cases):

```{language}
// Before (broken):
{problematic code pattern}

// After (fixed):
{corrected code with comment explaining the key change}
```

**Why this works**: {1–2 sentences explaining the mechanism, not just the syntax}

---

### If That Didn't Fix It

**Cause 2**: {Description of less common cause}

```{language}
{fix}
```

**Cause 3**: {Description of less common cause}

```{language}
{fix}
```

---

### Prevention

To prevent this class of error in the future:

```{language}
// Defensive pattern:
{preventive code pattern}
```

---

**Sources**:
- [{Primary source}]({URL}) — {why authoritative}
- [{Secondary source}]({URL}) — {corroborating}
```

---

## When to Escalate

Research can identify what needs fixing, but sometimes the real problem is implementation-specific. Know when to hand off.

### Escalate to Another Agent When:

| Situation | Escalate To |
|-----------|-------------|
| Error is in React component logic | react-expert |
| Error is in Python/FastAPI code structure | python-expert or fastapi-expert |
| Error is in database query/ORM | database-expert |
| Error is in Docker/container setup | docker-expert |
| Error reveals architectural mismatch | system-architect |
| Error is in CI/CD pipeline | devops-expert |
| Error is in TypeScript types | typescript-expert |

**Escalation language:**
```
"The research shows the root cause is {X}. This is now an implementation problem
rather than a research problem. The {agent-name} agent can implement the fix:
[summary of what needs to change]"
```

### Do Not Escalate When:

- The fix is a simple one-line change with a well-understood cause
- The user just needs to update a dependency version
- The error is caused by a typo or wrong import path

---

## Systematic Investigation Process

For difficult bugs where the error message alone is insufficient:

### Step 1 — Gather Complete Context

Before searching, collect:
- Full error message and stack trace (not just the first line)
- Library versions: runtime + all relevant dependencies
- What changed recently (last working state)
- Operating system and environment (Node version, Python version, OS)

### Step 2 — Isolate the Failure

Determine the smallest reproduction:
- Does the error occur in isolation or only in the full app context?
- Does it occur with minimal input or only specific inputs?
- Is it deterministic or intermittent?

### Step 3 — Search Strategy by Error Class

```
Runtime error → search exact error message + library name
Build error → search error code + build tool + version
Network error → search HTTP status code + library + operation
Permission error → search error code + OS + context
Version conflict → search "peer dependency" + packages involved
```

### Step 4 — Verify Solution Against Context

Before presenting a found solution, verify:
- Does the solution target the same version?
- Does the solution match the user's context (framework, architecture)?
- Does the solution have positive confirmation from multiple users?

---

## References

For error taxonomy with canonical search queries by error category, and diagnostic question checklists for complex bugs, see [error-analysis.md](references/error-analysis.md).

For how to search GitHub Issues, Stack Overflow operators, npm registry changelogs, and library migration guides for debugging information, see [debugging-sources.md](references/debugging-sources.md).
