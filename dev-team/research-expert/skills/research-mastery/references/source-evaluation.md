# Source Evaluation Reference

Deep-dive guide for evaluating the trustworthiness and currency of technical sources before using them in research findings.

---

## Trust Tier Matrix

| Source Type | Default Trust | Conditions That Raise Trust | Conditions That Lower Trust |
|-------------|--------------|----------------------------|----------------------------|
| Official docs | High | Version-specific URL, recently updated | Outdated version, community-maintained (not vendor) |
| Package registry (npm, PyPI) | High | Links to canonical GitHub repo | No source link, many open issues, no recent release |
| Library GitHub README | High | Actively maintained (recent commits) | Archived repo, stale issues >1 year |
| Library test files (`/test`, `/spec`) | High | Passes CI, tagged to release | Tests disabled, skipped widely |
| Maintainer GitHub Issues response | High | Maintainer has blue checkmark or "member" badge | Long gap between last activity |
| Stack Overflow accepted answer | Medium-High | 50+ upvotes, accepted, < 2 years old | Accepted but old with correction comments |
| Official framework blog | Medium-High | Posted by org account | Guest post with no org affiliation |
| Community blog (dev.to, Medium) | Medium | Author is known contributor, code verified in comments | Anonymous, no date, no version |
| YouTube tutorial | Low-Medium | Recent, high subscriber count | No date shown, uses deprecated API |
| Reddit / Discord / Slack | Low | Maintainer answers in official channel | Random community member |

---

## Staleness Detection

### Fast-Moving Ecosystems (Check < 1 Year)

These technologies change API surfaces frequently:

- React / Next.js / Remix
- Node.js (ESM vs CJS, built-in fetch, etc.)
- Python packaging (pyproject.toml, uv, ruff)
- Rust edition changes
- Kubernetes API versions
- AWS SDK v2 → v3 migrations

**Check**: Does the code use imports/patterns from the major version in the project?

### Slow-Moving Ecosystems (2–3 Years Acceptable)

Fundamentals are stable:

- SQL and database design patterns
- HTTP/REST API design principles
- Git workflows
- UNIX command-line tools
- Cryptography primitives

### Staleness Signals in Code

Scan code examples for these deprecated patterns before trusting them:

```
# JavaScript / Node.js staleness signals
require() instead of import (may be fine, but check)
callbacks instead of Promises/async (pre-2018 pattern)
var instead of const/let
webpack 4 config instead of 5
class components instead of React hooks (React < 16.8)
moment.js usage (deprecated in favor of date-fns/dayjs)

# Python staleness signals
print as statement (Python 2)
setup.py only, no pyproject.toml
urllib2 (Python 2)
asyncio.coroutine decorator (deprecated since 3.8)
typing.List/Dict instead of list/dict (pre-3.9)

# General staleness signals
No type annotations in a typed codebase
HTTP instead of HTTPS for API calls
MD5 for password hashing
```

---

## Version Verification Checklist

Before presenting any code example:

1. **Identify the version** — what version does the example target?
   ```
   Search: "{library} {code-snippet-keyword} changelog" to find when it was introduced
   ```

2. **Compare to project version** — what version does the user's project use?
   ```
   Check: package.json, requirements.txt, Cargo.toml, go.mod
   ```

3. **Check for removal** — was this API removed in a version between the example's target and the project's version?
   ```
   Search: "{library} CHANGELOG.md v{version} removed deprecated"
   ```

4. **Check for breaking renames** — same concept, different method name?
   ```
   Search: "{library} migration guide v{old} to v{new}"
   ```

---

## Red Flag Identification Flowchart

```
Source found →
  Does it have a version number?
    NO → Search for version-specific alternative
    YES → Does version match project?
            NO → Check changelog for compatibility
            YES → Continue

  Does it have a publication date?
    NO → Check Wayback Machine or git blame for approximate date
         If > 2 years for fast ecosystem → deprioritize

  Does the code use imports that exist in current API?
    Check official API reference or GitHub source
    NO → Find current equivalent or mark as needs-adaptation

  Do comments/other answers confirm it works?
    NO → Test mentally line-by-line or note "unverified"
```

---

## Official Source Locators by Ecosystem

### Finding Official Docs

| Ecosystem | Official Doc Pattern | Registry |
|-----------|---------------------|----------|
| npm package | github.com/{owner}/{repo}/blob/main/README.md | npmjs.com/{pkg} |
| PyPI package | pypi.org/project/{pkg} → Homepage link | readthedocs.io |
| Rust crate | docs.rs/{crate} | crates.io |
| Go module | pkg.go.dev/{module} | pkg.go.dev |
| Ruby gem | rubydoc.info/gems/{gem} | rubygems.org |

### Finding Changelogs

Most projects follow one of these conventions:
- `CHANGELOG.md` or `CHANGES.md` in root of repo
- GitHub Releases tab: `github.com/{owner}/{repo}/releases`
- Dedicated migration guide: `docs.{framework}.org/migration`
- Breaking change labels in GitHub Issues

---

## Citation Standards

When presenting research findings, cite sources consistently:

```markdown
**Source**: [{Short Title}]({URL})
**Type**: Official documentation / Accepted SO answer / Maintainer response
**Version**: {library} v{X.Y}
**Date**: {YYYY-MM or "current" if URL is versioned}
**Confidence**: High / Medium / Low
```

For multiple sources that agree, list the primary and note others corroborate:

```markdown
**Primary source**: [Express docs — Rate limiting](https://expressjs.com/...) (Official, v4.x)
**Corroborated by**: 3 SO answers, express-rate-limit README examples
```
