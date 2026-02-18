# Technology Trends Reference

Metrics, sources, and frameworks for assessing the health, adoption trajectory, and maturity of any technology or library.

---

## Health Metrics by Platform

### npm Packages

**Where to check:**

| Metric | Source | Healthy Signal |
|--------|--------|---------------|
| Weekly downloads | npmjs.com/package/{name} | Stable or growing trend |
| Download trend | npmtrends.com | Not declining sharply |
| Last publish | npmjs.com/package/{name} | Within 6 months for active libs |
| Version count | npmjs.com/package/{name}?activeTab=versions | Regular releases |
| Dependent packages | npmjs.com/package/{name}?activeTab=dependents | Many dependents = ecosystem confidence |
| Bundle size | bundlephobia.com/package/{name} | < 50kB gzipped for most utilities |

**Interpreting download numbers:**

| Downloads/week | Interpretation |
|---------------|----------------|
| > 10M | Ecosystem infrastructure (used by tools) |
| 1M–10M | Dominant in its category |
| 100K–1M | Well-established, active community |
| 10K–100K | Viable choice, smaller community |
| < 10K | Niche or declining |

**Caution:** Download counts include CI pipelines and automated installs. A package can have 1M downloads/week and still be declining if it's only installed by legacy build tooling.

### PyPI Packages

**Where to check:**

```
https://pypistats.org/packages/{package-name}
```

| Metric | Where | Healthy Signal |
|--------|-------|---------------|
| Daily downloads | pypistats.org | Stable or growing |
| Download trend | pypistats.org → chart | 30-day rolling average not declining |
| Latest version date | pypi.org/project/{name} | Recent for active projects |
| Supported Python versions | pypi.org/project/{name} | Includes current Python version |

### GitHub Repository Signals

**For any GitHub-hosted project:**

```
Check at: github.com/{owner}/{repo}

Stars:           High absolute count + upward trend
Forks:           High fork count = active contributors / derivatives
Open issues:     < 15% of total issues = well-maintained
PRs merged/week: Check Insights > Pulse tab
Contributors:    Multiple contributors = not bus-factor-1
Last commit:     Main branch — within 30 days for active projects
```

**Star history (trend matters more than count):**
```
https://star-history.com/#owner/repo
```

**Red flags:**
- `[UNMAINTAINED]` or `[DEPRECATED]` in repository description or README header
- Issues labeled "stale" with no maintainer response for > 3 months
- Last release > 1 year ago with open issues and PRs not being merged
- Repository archived (GitHub shows "This repository has been archived")

---

## Ecosystem Maturity Signals

### Mature Ecosystem Indicators

A technology is mature and lower-risk to adopt when:

1. **Multiple competing implementations** exist (not a monopoly)
2. **IDE support** (LSP, extensions for major editors)
3. **Testing utilities** are published and maintained
4. **Commercial backing** or foundation support (CNCF, Apache, Linux Foundation)
5. **Published CVEs and patches** — security vulnerabilities are found AND fixed (no patches = no users looking OR no maintainers responding)
6. **Migration guides** from older versions — team has thought about backward compatibility
7. **Benchmarks published** — team measures and cares about performance
8. **API stability guarantees** — documented deprecation policy

### Early-Stage Warning Signs

Be cautious when:

1. **Only one maintainer** with no organizational backing
2. **No stable release** (still at v0.x)
3. **Large number of open issues with no response** in last 3 months
4. **Changelog shows many breaking changes** between minor versions
5. **Documentation is sparse** — only a README, no full docs site
6. **Test coverage low or not published**

---

## Where to Find Trend Data

### For Comparing Technologies

| What to Compare | Best Source |
|----------------|-------------|
| npm packages | npmtrends.com/{pkg-a}-vs-{pkg-b} |
| Python packages | pypistats.org (individual) |
| General tech adoption | survey.stackoverflow.com/2024 (Stack Overflow annual survey) |
| Framework popularity | stateofjs.com (JavaScript), stateofpython.com |
| General web trends | w3techs.com (usage across websites) |
| DevOps tools | dora.dev (DORA metrics report) |

### Stack Overflow Developer Survey

Published annually (May): survey.stackoverflow.com/results/{year}

Key sections:
- **Most Popular Technologies** — what developers are using
- **Most Loved / Dreaded** — sentiment, predicts future adoption
- **Most Wanted** — what developers want to learn (predicts future adoption)
- **Salary correlations** — indicates professional/enterprise adoption

### State of JS / State of CSS / State of Python

Annual community surveys:
- stateofjs.com — JavaScript ecosystem deep-dive
- stateofcss.com — CSS tools and features
- lp.jetbrains.com/python-developers-survey-{year}/ — Python ecosystem

### TIOBE / PYPL Indices

- tiobe.com/tiobe-index/ — Programming language popularity (search engine query volume)
- pypl.github.io — Popularity of Programming Language (tutorial searches)

These are lagging indicators (historical) but useful for language-level decisions.

---

## Red Flags for Abandoned Projects

If a project shows 3+ of these signals, it should be considered abandoned or at high risk:

```
[ ] Last commit to main branch > 6 months ago
[ ] Open issues with maintainer questions unanswered for > 3 months
[ ] Open PRs from contributors not merged for > 3 months
[ ] Issues labeled "stale" with bot auto-closing (no human involvement)
[ ] README not updated to reference current version numbers
[ ] CI pipeline failing on main branch (visible via badge in README)
[ ] Latest version fails to install cleanly with current toolchain
[ ] GitHub discussion/Issues show users migrating to alternatives
[ ] npm/PyPI page shows "This package has not been updated in a long time"
[ ] Repository transferred or renamed without forwarding
```

**Action when project appears abandoned:**
1. Search for forks that are actively maintained: `github.com/{owner}/{repo}/network/members` (sort by recently updated)
2. Search for official replacements: "{project} alternative maintained" or check the repo for pinned issue about successors
3. Check if the functionality has been absorbed into a larger ecosystem (e.g., moment.js → Temporal / date-fns)

---

## Ecosystem-Specific Trend Sources

### JavaScript / TypeScript

- **Twitter/X**: @ThePrimeagen, @t3dotgg, @wesbos, @cassidoo — high signal on current trends
- **Bytes newsletter**: bytes.dev — weekly JavaScript news
- **JS Weekly**: javascriptweekly.com — weekly curated
- **Node.js**: nodejs.org/en/blog — official releases and deprecations

### Python

- **Python Insider**: blog.python.org — official CPython news
- **PyCon talks**: pyvideo.org — conference talks on ecosystem trends
- **Python Packaging**: packaging.python.org/en/latest/ — authoritative packaging guidance

### Rust

- **This Week in Rust**: this-week-in-rust.org — weekly ecosystem news
- **Rust Blog**: blog.rust-lang.org — official edition and feature announcements

### Go

- **Go Blog**: go.dev/blog — official language updates
- **GopherCon**: conference talks on ecosystem direction

### DevOps / Cloud

- **CNCF Annual Survey**: cncf.io/reports/cncf-annual-survey — cloud-native adoption
- **ThoughtWorks Tech Radar**: thoughtworks.com/radar — hold/assess/trial/adopt classification
