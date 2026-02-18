---
name: Research Mastery
triggers:
  - "research this"
  - "find documentation"
  - "look up"
  - "find information about"
  - "how does X work"
  - "what is the best way"
  - "research official docs"
description: |
  This skill should be used when the research-expert needs foundational methodology for any research task — establishing source hierarchy, constructing effective queries, evaluating source quality, and synthesizing multiple sources into coherent developer guidance. It covers the core workflow that underlies all other research skills.
---

# Research Mastery: Methodology, Source Evaluation, and Synthesis

## Research Hierarchy

Not all sources are equal. Search them in order, stopping when you have sufficient verified information.

### Tier 1 — Official Sources (Highest Trust)

These sources are authoritative by definition:

- **Official documentation sites** — docs.python.org, reactjs.org, docs.rs, etc.
- **Framework changelogs and migration guides** — definitive on breaking changes
- **Package registry pages** — npmjs.com, pypi.org, crates.io, pkg.go.dev (for health signals: weekly downloads, latest version, license, repository link)
- **Library's own GitHub repository** — especially `examples/`, `test/`, and `README.md`

Always prefer the docs for the specific version in use. Use URL patterns like `docs.example.com/v2.x/` rather than unversioned URLs.

### Tier 2 — Authoritative Community (High Trust)

These sources are reliable when official docs are incomplete:

- **Stack Overflow** — filter for accepted answers (green checkmark) with 10+ votes, posted within the last 2 years for fast-moving ecosystems
- **GitHub Issues and Discussions** — maintainer-answered threads are nearly as reliable as docs
- **Official blog posts** from the library/framework team (check author affiliation)

### Tier 3 — Community Content (Moderate Trust, Verify First)

Use only when Tier 1–2 sources are insufficient:

- **Dev.to, Medium, Hashnode blog posts** — check: author credentials, publication date, whether code has been tested by commenters
- **YouTube tutorials** — useful for understanding flow, but verify code against current API
- **Reddit, Discord** — only for identifying what to search, never as a primary source

### Tier 4 — Deprecated / Avoid

- Blog posts older than 2 years for fast-moving ecosystems (React, Node.js, Python packaging)
- Unofficial tutorials with no version tags
- StackOverflow answers marked "outdated" by comments or with negative votes

---

## Query Construction

Effective queries retrieve specific, version-aware results.

### Basic Anatomy of a Good Query

```
{library} {version} {specific-feature} {language?} example
```

**Examples:**
- `express rate limiting middleware v4 example` — specific version, specific feature
- `react useEffect dependency array warning fix 2024` — date disambiguates modern advice
- `python asyncio gather error handling TypeError site:stackoverflow.com` — site filter for quality

### Query Patterns by Task Type

**Finding official docs:**
```
site:docs.{framework}.{tld} {feature}
{library} official documentation {feature}
```

**Finding code examples:**
```
{library} {feature} example github
{function name} usage example site:github.com
```

**Debugging errors:**
```
"{exact error message}" {library} {version}
{error type} {library} fix site:stackoverflow.com
```

**Comparing options:**
```
{option-a} vs {option-b} {use-case} 2024
{option-a} OR {option-b} comparison {framework} pros cons
```

**Finding migration guides:**
```
{library} migrate v{old} to v{new}
{library} breaking changes {version} changelog
```

### Query Refinement Strategy

If the first query returns poor results:

1. Add version number — `express 4.x` narrows better than just `express`
2. Add the error's exact phrasing in quotes — `"Cannot read property of undefined"`
3. Narrow to site — `site:github.com`, `site:stackoverflow.com`
4. Add language — `"python 3.12"` or `"node 20"`
5. Add recency — `after:2023` in Google, or manually filter by date

---

## Source Evaluation Framework

Before using a source, run it through this checklist:

### Green Flags (Trust Indicators)

- Version number matches what's in use (or is compatible)
- Publication or last-updated date is recent (within 1–2 years for active ecosystems)
- Author is the library maintainer or has verifiable expertise
- Code examples have been tested (commenters confirm, or examples are from test suites)
- Multiple independent sources agree on the same approach
- Official repo links to or cites the source

### Red Flags (Distrust Indicators)

- No version mentioned anywhere in the article
- No publication date visible
- Code uses APIs that no longer exist (check against current changelog)
- Many negative comments or corrections below
- The only source saying this approach — find corroboration
- Written in a tone optimized for SEO ("10 best ways to...") rather than accuracy

### Version Staleness Check

For any code example, verify these imports/calls still exist in the current version:

```bash
# For npm packages: check version at time of writing vs current
# Search: "{package} CHANGELOG.md" or "{package}/releases on GitHub"
# Look for: removed APIs, renamed functions, changed signatures
```

---

## Synthesis and Output Format

Raw search results are never the final deliverable. Always synthesize.

### The Synthesis Process

1. **Read all sources** collected from the priority tiers above
2. **Identify consensus** — what do 2+ authoritative sources agree on?
3. **Identify conflicts** — where do sources disagree? Note why (different versions, different use-cases)
4. **Extract the essential** — distill to: the recommendation, the rationale, and one working code example
5. **Format for immediate use** — developer-ready, not raw notes

### Structured Output Template

```markdown
## Research Findings: {Topic}

**Recommendation**: {One-sentence bottom line with confidence level}

**Why**: {2–3 sentences explaining rationale, citing key sources}

### Implementation

```{language}
// Working code example — tested against {library} vX.Y
// Key parameter: {explain what to change}
{code}
```

### Configuration Options

| Option | Default | Effect |
|--------|---------|--------|
| {name} | {value} | {description} |

### Alternatives

- **{Alternative A}**: Choose when {condition}. Trade-off: {what you gain/lose}.
- **{Alternative B}**: Choose when {condition}. Trade-off: {what you gain/lose}.

### Sources

- [{Source Title}]({URL}) — {one sentence on why this source was used}
```

### Handling Conflicting Sources

When sources disagree:

1. **Check versions first** — most conflicts are actually version differences
2. **Defer to official docs** — when official docs and community blog disagree, trust official docs
3. **Note the conflict explicitly** — tell the developer "Source A says X, Source B says Y; they differ because..."
4. **Present both** when both are valid for different scenarios

---

## Documentation Curation

Save research findings to `PRPs/ai_docs/` when:

- The library is complex enough that research took > 5 minutes
- The findings involve non-obvious gotchas specific to this project's stack
- The technology will likely be used again in this project

**Save format:**

```markdown
# {Library} Patterns — Researched {YYYY-MM-DD}

## Version
{library} vX.Y.Z

## Key Patterns

### {Pattern Name}
{code example}
{why and when to use}

## Gotchas
- {issue}: {solution}

## Sources
- {URL} — {why authoritative}
```

Save to: `PRPs/ai_docs/{library-name}-patterns.md`

---

## References

For detailed source trust evaluation, staleness detection, and red-flag identification, see [source-evaluation.md](references/source-evaluation.md).

For synthesis templates, conflict-resolution patterns, and output formats for different request types (how-to, comparison, debugging, architecture decision), see [synthesis-patterns.md](references/synthesis-patterns.md).
