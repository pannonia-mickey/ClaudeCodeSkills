---
name: Research Best Practices
triggers:
  - "best practices"
  - "recommended approach"
  - "modern way to"
  - "what should I use"
  - "which is better"
  - "industry standard"
  - "current standard"
  - "idiomatic"
description: |
  This skill should be used when the research-expert needs to identify current best practices, recommended patterns, or the leading option among alternatives for any technology, library, or architectural approach. It covers how to find community consensus, assess technology health and recency, compare competing options, and structure a recommendation with clear rationale.
---

# Research Best Practices: Identifying Current Idiomatic Approaches

## Community Consensus Signals

Best practices are not opinions — they are verified by measurable community signals. Assess these signals before making a recommendation.

### Primary Signals (Most Reliable)

**1. Official Recommendation**

The framework/language's own documentation recommending a pattern or library is the strongest possible signal.

```
Search: "{framework} official recommendation {topic}"
Check: Does the official docs page link to or mention the library?
Example: React docs recommending Vite for new projects (over Create React App)
```

**2. Download/Usage Volume Trends**

High downloads indicate widespread adoption, but trends matter more than absolute numbers.

```
# npm trends:
https://npmtrends.com/{pkg-a}-vs-{pkg-b}-vs-{pkg-c}

# PyPI stats:
https://pypistats.org/packages/{package}

# Bundlephobia (bundle size matters for frontend):
https://bundlephobia.com/package/{package}
```

Look for: Is the trend line going up or down over the last 12 months?

**3. GitHub Repository Health**

```
Check on github.com/{owner}/{repo}:
- Stars: absolute count + trend (use star-history.com/{owner}/{repo})
- Open issues vs closed issues ratio (high open/closed = poor maintenance)
- Last commit date (> 6 months without commit = concerning for active development)
- Release frequency (regular releases = active maintenance)
- Response time on issues (maintainers active = good support)
```

**4. Stack Overflow Tag Activity**

```
https://stackoverflow.com/tags/{tag}/info
```

Growing question volume + high answer rate = active community.

### Secondary Signals (Supporting Evidence)

- **Conference talks** — Libraries featured at major conferences (ReactConf, PyCon, GopherCon) have community validation
- **Inclusion in official starter templates** — e.g., `create-next-app` including Tailwind as an option = strong endorsement
- **Corporate adoption** — Used by Vercel, Meta, Google, Microsoft (check their engineering blogs or open-source repos)

---

## Recency Assessment

Best practices become outdated. Before recommending, verify recency.

### Fast-Moving Ecosystems (Reassess Every 6–12 Months)

These change fastest:

| Ecosystem | What Changes | How to Check Recency |
|-----------|-------------|---------------------|
| JavaScript/TypeScript | Bundlers (webpack→vite→turbopack), meta-frameworks | npm trends, Theo/t3 stack changes |
| Python packaging | pip→poetry→uv, setup.py→pyproject.toml | Python packaging user survey |
| React | State management, data fetching patterns | react.dev blog, React team tweets |
| Cloud/Infrastructure | Provider-native vs third-party tools | AWS/GCP/Azure blog posts |

### Checking for Deprecation or Supersession

```
Search: "{library} deprecated" OR "{library} replaced by" OR "{library} abandoned"
Check: README for deprecation notices
Check: package.json/pyproject.toml for "deprecated" status on registry page
Search: "{library} alternative 2024" — see what the community recommends as replacement
```

### Migration Guide Existence = Best Practice Has Evolved

If a library has a migration guide from v1 to v2, the v1 patterns may now be anti-patterns:

```
Search: "{library} migration guide"
github.com/{owner}/{repo}/blob/main/MIGRATION.md
```

---

## Comparison Framework

Use this framework when evaluating multiple competing options.

### Evaluation Dimensions

| Dimension | How to Assess | Why It Matters |
|-----------|--------------|----------------|
| **Correctness** | Official recommendation? Community consensus? | Primary criterion |
| **Bundle size** | bundlephobia.com | Frontend performance |
| **TypeScript support** | Native types vs @types/ vs none | DX and safety |
| **Active maintenance** | Commits in last 90 days, open issue count | Long-term viability |
| **Learning curve** | Community perception, tutorial quality | Team adoption speed |
| **Breaking change frequency** | Major version history | Migration cost |
| **Community size** | SO questions, Discord/Slack members | Support availability |
| **Performance** | Published benchmarks (check recency and methodology) | Production viability |
| **License** | MIT/Apache = permissive; GPL = copyleft; custom = check | Legal compliance |

### Comparison Research Process

1. **List candidates** — what are the 2–4 realistic options?
2. **Find each option's pitch** — why does each option claim to be better?
3. **Find critical comparisons** — who has done head-to-head comparisons?
   ```
   Search: "{option-a} vs {option-b} {use-case}"
   Search: "{topic} comparison {year}"
   ```
4. **Check official positions** — does the framework recommend one over the others?
5. **Check community sentiment** — what do recent Reddit/HN discussions say?
   ```
   site:reddit.com/r/{relevant-subreddit} {option-a} vs {option-b}
   site:news.ycombinator.com {option-a}
   ```

---

## Recommendation Structure

Always deliver a recommendation with clear rationale, not a neutral "it depends" non-answer.

### Standard Recommendation Format

```markdown
## Recommendation: {Chosen Option}

**Bottom line**: Use {chosen option} for {use-case}. {2-sentence rationale grounded in evidence}.

### Why {Chosen Option} Wins for This Case

- **{Criterion 1}**: {concrete evidence — e.g., "4M weekly npm downloads vs 200K for alternative"}
- **{Criterion 2}**: {concrete evidence — e.g., "Official React docs recommend it in the data fetching guide"}
- **{Criterion 3}**: {concrete evidence — e.g., "Native TypeScript types, no @types/ package needed"}

### When to Choose {Alternative} Instead

Use {alternative} when:
- {Specific condition where alternative wins}
- {Specific condition where alternative wins}

### Getting Started

```{language}
// Minimal working example with {chosen option}
// {library} v{X.Y}
{code}
```

### Sources
- [{Official docs section}]({URL}) — official recommendation
- [npm trends comparison]({URL}) — download volume data
- [{Community comparison}]({URL}) — head-to-head analysis
```

### Avoid "It Depends" Non-Answers

"It depends" is only acceptable when you follow it immediately with the specific conditions:

```
# Unacceptable:
"It depends on your use case."

# Acceptable:
"It depends: use Zustand if your global state is simple (< 5 interconnected stores),
use Jotai if you need fine-grained atom-level reactivity, and use Redux Toolkit
if you're already in a Redux codebase or need time-travel debugging."
```

---

## Identifying Outdated Patterns

These patterns appear in older tutorials but are no longer recommended:

### JavaScript / Node.js

| Outdated Pattern | Modern Replacement | Signal |
|-----------------|-------------------|--------|
| `var` declarations | `const`/`let` | ES2015+ |
| Callback hell | `async/await` | Node 8+ |
| `moment.js` | `date-fns` or `Temporal` | Moment deprecated 2020 |
| `request` package | `node-fetch`, `axios`, or native `fetch` | request deprecated 2020 |
| Webpack 4 config | Vite or Webpack 5 | 2021+ |
| Create React App | Vite + React template | CRA deprecated 2023 |
| Redux boilerplate | Redux Toolkit | RTK is now the official approach |
| `enzyme` for React testing | React Testing Library | RTL is current standard |

### Python

| Outdated Pattern | Modern Replacement | Signal |
|-----------------|-------------------|--------|
| `setup.py` only | `pyproject.toml` | PEP 518, 2018+ |
| `pipenv` | `uv` or `poetry` | uv gaining rapidly in 2024 |
| `print` debugging | `logging` or `rich` | Any production Python |
| `os.path.join` | `pathlib.Path` | Python 3.4+ |
| `typing.List[str]` | `list[str]` | Python 3.9+ |
| `django-rest-framework` for new APIs | FastAPI + Pydantic | For new API-only projects |

### CSS / Frontend

| Outdated Pattern | Modern Replacement | Signal |
|-----------------|-------------------|--------|
| CSS-in-JS (styled-components, emotion) | Tailwind CSS | Bundle size, performance |
| jQuery | Vanilla JS or framework | Any modern browser |
| Sass/LESS | CSS custom properties + PostCSS | Modern CSS features |
| Placeholder selectors | Native CSS nesting | CSS Nesting spec, 2023+ |

---

## References

For metrics to check technology health, npm/GitHub signals, and ecosystem maturity indicators, see [technology-trends.md](references/technology-trends.md).

For a reference library of "current standard vs. outdated pattern" across API design, authentication, error handling, testing, CI/CD, and security categories, see [practice-patterns.md](references/practice-patterns.md).
