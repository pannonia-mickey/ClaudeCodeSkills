# Synthesis Patterns Reference

Templates and patterns for transforming raw research findings into coherent, developer-ready guidance.

---

## Output Templates by Request Type

Different research requests require different synthesis shapes.

### How-To Request

> "How do I implement X?"

```markdown
## How to {Task}

**Quick answer**: {One sentence — the simplest correct approach}

### Setup

```{language}
// Step 1: Installation or prerequisite
{code}
```

### Implementation

```{language}
// Working example — {library} v{X.Y}
{code with inline comments explaining non-obvious lines}
```

### What This Does

{2–3 sentence explanation of the mechanism, not just the syntax}

### Common Variations

- **{Variation A}**: `{code snippet}` — use when {condition}
- **{Variation B}**: `{code snippet}` — use when {condition}

### Watch Out For

- {Gotcha 1}: {brief explanation and fix}
- {Gotcha 2}: {brief explanation and fix}

**Sources**: [{Title}]({URL})
```

---

### Comparison Request

> "What's better: X or Y?" / "Which library should I use?"

```markdown
## {Option A} vs {Option B} for {Use Case}

**Recommendation**: Use **{winner}** when {conditions that apply to this developer}.
Use **{alternative}** when {conditions for choosing alternative}.

### Quick Comparison

| Dimension | {Option A} | {Option B} |
|-----------|------------|------------|
| Setup complexity | {Low/Med/High} | {Low/Med/High} |
| Bundle size | {size} | {size} |
| Active maintenance | {Yes/No — last release date} | {Yes/No} |
| TypeScript support | {Native/Via @types/None} | {Native/Via @types/None} |
| {Relevant dimension} | {value} | {value} |

### {Option A} — When to Choose

**Best for**: {ideal use case}

```{language}
// Minimal working example
{code}
```

**Pros**: {2–3 bullet points}
**Cons**: {1–2 bullet points}

### {Option B} — When to Choose

**Best for**: {ideal use case}

```{language}
// Minimal working example
{code}
```

**Pros**: {2–3 bullet points}
**Cons**: {1–2 bullet points}

**Sources**: [{Option A} docs]({URL}), [{Option B} docs]({URL})
```

---

### Debugging Request

> "I'm getting error X" / "Why is this failing?"

```markdown
## Debugging: {Error Message or Symptom}

**Root Cause**: {One sentence — what actually causes this error}

**Most Common Fix**:

```{language}
// Before (broken)
{broken code}

// After (fixed)
{fixed code with comment explaining the change}
```

### Why This Happens

{2–3 paragraph explanation of the mechanism, enough to prevent recurrence}

### Other Causes (If Above Didn't Fix It)

**Cause 2**: {description}
```{language}
{fix}
```

**Cause 3**: {description}
```{language}
{fix}
```

### Prevention

```{language}
// Defensive pattern to avoid this error class
{code}
```

**Sources**: [{Stack Overflow answer}]({URL}), [{Library issue}]({URL})
```

---

### Architecture Decision Request

> "What's the best way to structure X?" / "Should I use pattern A or B?"

```markdown
## Architecture Decision: {Decision}

**Recommended Approach**: {name}

**Decision Factors for This Case**:
- {Factor 1}: {why it favors the recommendation}
- {Factor 2}: {why it favors the recommendation}
- {Factor 3}: {any concerns and how recommendation addresses them}

### Recommended: {Approach A}

```{language}
// Skeleton of recommended architecture
{code}
```

**Why**: {2–3 sentences grounded in principles, not opinion}

**Trade-offs**: {what you give up — be honest}

### Alternative: {Approach B}

Prefer this when: {specific conditions that make it better}

**Trade-offs vs recommendation**: {brief}

### Decision Criteria Matrix

| Criteria | Recommended Wins | Alternative Wins |
|----------|-----------------|-----------------|
| {Criteria 1} | ✓ | |
| {Criteria 2} | | ✓ |
| {Criteria 3} | ✓ | |

**Sources**: [{Architecture reference}]({URL}), [{Case study}]({URL})
```

---

## Conflict Resolution

When multiple authoritative sources disagree, use this resolution process:

### Step 1: Version Check

Most conflicts are version differences, not actual disagreements.

```
Source A says: use {approach A}
Source B says: use {approach B}

Check: What version does each source target?
If different versions → Both may be correct for their respective versions
```

### Step 2: Recency + Authority Ranking

If same version, rank by authority:
1. Official docs (current version)
2. Maintainer response in GitHub Issues
3. Accepted SO answer with 50+ votes
4. High-quality blog post

The higher-ranked source wins the conflict.

### Step 3: Explicit Conflict Presentation

When both approaches are valid in different contexts, present both:

```markdown
**Note — Sources Disagree**

**Source A** ([{Title}]({URL})) recommends {approach A}.
This is appropriate when: {conditions A}.

**Source B** ([{Title}]({URL})) recommends {approach B}.
This is appropriate when: {conditions B}.

**For your case** ({relevant context}): use {recommendation} because {reason}.
```

---

## Progressive Summarization

When synthesizing multiple long documents into a short response:

### Layer 1 — The Bottom Line (1 sentence)

Lead with the answer, not the research:
> "Use `express-rate-limit` with a Redis store for distributed rate limiting."

### Layer 2 — The Rationale (1 paragraph)

Why this answer is correct, and the key evidence:
> "express-rate-limit is the most-downloaded option (4M weekly downloads), actively maintained, and has first-class Redis support via `rate-limit-redis`. The official Express docs recommend it in their security best practices guide."

### Layer 3 — The Code (1 working example)

The minimal working example, commented:
```javascript
// express-rate-limit v7 — Redis store for distributed environments
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis').default;

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15-minute window
  max: 100,                    // requests per window per IP
  standardHeaders: 'draft-7',  // Return RateLimit headers
  store: new RedisStore({
    sendCommand: (...args) => redisClient.sendCommand(args),
  }),
});

app.use('/api/', limiter);
```

### Layer 4 — The Depth (on request)

Only provide configuration tables, alternatives, and gotchas if the developer asks for more, or if critical gotchas would block implementation.

---

## Multi-Source Aggregation Pattern

When combining information from 4+ sources:

1. **Gather all sources** into a mental table: {source → key claim → confidence}
2. **Find consensus claims** — appears in 2+ authoritative sources → include as fact
3. **Find contested claims** — only in 1 source, or sources disagree → flag explicitly
4. **Find unique gotchas** — a single source mentions a critical edge case → include with source citation

```markdown
### Implementation Notes (Aggregated from 5 sources)

✓ **Consensus**: {claim} (confirmed by {Source A}, {Source B}, {Source C})
⚠ **Contested**: {claim} ({Source A} says X, {Source B} says Y — see note below)
! **Critical gotcha** from [{Source D}]({URL}): {specific issue} — affects {scenario}
```
