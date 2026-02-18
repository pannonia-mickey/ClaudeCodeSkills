# Debugging Sources Reference

Where to find debugging information by error type, platform, and ecosystem — with specific search syntax, URL patterns, and navigation guides.

---

## GitHub Issues Search

GitHub Issues is the best source for library-specific bugs, especially errors that don't appear on Stack Overflow.

### Search Syntax

```
# Basic search within a repo
github.com/{owner}/{repo}/issues?q={search-terms}

# Advanced: search for specific error text
github.com/{owner}/{repo}/issues?q="exact error message"

# Filter by state
github.com/{owner}/{repo}/issues?q={terms}&state=all     # Open + closed
github.com/{owner}/{repo}/issues?q={terms}&state=closed  # Closed issues often have solutions

# Filter by label
github.com/{owner}/{repo}/issues?q={terms}+label:bug
```

### Interpreting GitHub Issue Results

| Signal | Meaning |
|--------|---------|
| Issue is closed with "fixed" label | A release fixed this — check which version |
| Issue is closed with "wontfix" | Intentional behavior — need a workaround |
| Maintainer replies with code | Most reliable solution available |
| Issue says "duplicate of #X" | Follow the link to the original issue |
| Issue has many "+1" reactions | Common problem — more likely to have a proper fix |
| Issue is still open with no response | Known bug, no fix yet — look for workarounds in comments |

### Finding the Fix Version

When a GitHub Issue is closed as fixed:

```
1. Find the PR that fixed it: look for "Closed by #X" or "Fixed by #X" link
2. Click the PR → look for "Merged into" or check the milestone
3. Check which tag/release the PR is included in:
   github.com/{owner}/{repo}/releases → search for the PR number
4. If project version ≥ fix version → update the library
   If project version < fix version → apply workaround until you can update
```

---

## Stack Overflow Search Operators

### Basic Search

```
https://stackoverflow.com/search?q={query}
```

### Effective Query Patterns

```bash
# Tagged search (most useful — constrains to specific technology)
[express] [rate-limit] unexpected behavior
[python-3.11] [asyncio] TypeError

# Search for exact error message (use quotes)
"Cannot read properties of undefined" [react]

# Search with multiple tags
[node.js] [webpack] "MODULE_NOT_FOUND"

# Filter to answered questions only
[react] useEffect undefined state isanswered:1

# Sort by votes (finds most community-validated answers)
https://stackoverflow.com/search?q=[{tag}]+{keywords}&tab=Votes
```

### Evaluating a Stack Overflow Answer

```
Score ≥ 20         → Strong community endorsement
Accepted + Score   → Both asker and community verified
Recent date        → Relevant for fast-moving ecosystems
Top comments       → Check if comments update or correct the answer
Low score + accepted → Asker may have accepted a mediocre answer; read carefully
```

### Finding the Right Stack Overflow Question

When your search returns too many results:

1. **Filter by tag** — add the library name as a tag: `[{library-name}]`
2. **Add the exact error text** in quotes
3. **Filter by score** — change `tab=Newest` to `tab=Votes`
4. **Check "Linked" and "Related"** sidebar on a question — often finds better duplicate

---

## npm Registry Changelog Navigation

### Finding What Changed Between Versions

```
# Method 1 — GitHub Releases (most comprehensive):
github.com/{owner}/{repo}/releases

# Method 2 — CHANGELOG.md in repo:
github.com/{owner}/{repo}/blob/main/CHANGELOG.md

# Method 3 — npm registry version history:
npmjs.com/package/{package-name}?activeTab=versions
```

### Reading a CHANGELOG for Debugging Clues

Look for these keywords when a breakage happened after an update:

```
BREAKING CHANGE    → API changed, migration required
deprecated         → Feature removed in this or future version
removed            → Previously deprecated feature is now gone
renamed            → Function/class/option has a new name
fix                → May indicate your bug was already reported and fixed
```

### Finding a Specific Version's Code

To see what the library code looked like at a specific version:

```
github.com/{owner}/{repo}/blob/v{X.Y.Z}/{file-path}
```

Or browse all tags:

```
github.com/{owner}/{repo}/tags
```

---

## Python-Specific Debugging Sources

### PyPI Project Page

```
https://pypi.org/project/{package-name}/
```

Contains: version history (useful for identifying when a bug was introduced), project links (GitHub, docs), Python version compatibility.

### ReadTheDocs

Most Python packages host docs at:
```
https://{package-name}.readthedocs.io/en/stable/
```

The **Changelog** section is often at:
```
https://{package-name}.readthedocs.io/en/stable/changelog.html
```

### Python Bug Tracker (CPython)

For errors that look like Python runtime bugs (not library bugs):
```
https://bugs.python.org/
```

### Pip and Packaging Issues

For installation errors, the authoritative source is:
```
https://packaging.python.org/en/latest/
https://pip.pypa.io/en/stable/
```

Common `pip install` error sources:
```
# Dependency conflict
pip install {package} -v  # Verbose output shows dependency tree

# Find conflicting versions
pip-check                  # If installed: pip install pip-check
pipdeptree                 # If installed: pip install pipdeptree

# Force resolution
pip install {package} --upgrade --upgrade-strategy eager
```

---

## Community Support Channels by Ecosystem

### JavaScript / TypeScript

| Channel | Best For | Link |
|---------|----------|------|
| Stack Overflow `[javascript]`, `[node.js]`, `[react]` | Specific errors | stackoverflow.com |
| Reactiflux Discord | React-specific | discord.gg/reactiflux |
| Node.js Discord | Node runtime issues | discord.gg/nodejs |
| Vercel Discord | Next.js and deployment | discord.gg/vercel |
| GitHub Discussions (per project) | Library-specific | github.com/{owner}/{repo}/discussions |

### Python

| Channel | Best For | Link |
|---------|----------|------|
| Stack Overflow `[python]` | Specific errors | stackoverflow.com |
| Python Discord | General Python | discord.gg/python |
| Real Python | Learning-oriented | realpython.com |
| FastAPI GitHub Discussions | FastAPI-specific | github.com/tiangolo/fastapi/discussions |
| Django Forum | Django-specific | forum.djangoproject.com |

### Rust

| Channel | Best For | Link |
|---------|----------|------|
| Stack Overflow `[rust]` | Specific errors | stackoverflow.com |
| The Rust Users Forum | General questions | users.rust-lang.org |
| Rust Discord | Active community | discord.gg/rust-lang |
| docs.rs | API reference | docs.rs/{crate} |

### Go

| Channel | Best For | Link |
|---------|----------|------|
| Stack Overflow `[go]` | Specific errors | stackoverflow.com |
| Gopher Slack | Community | gophers.slack.com |
| Go Forum | General questions | forum.golangbridge.org |

---

## Stack Trace Extraction Guide

### Reading a Node.js Stack Trace

```
TypeError: Cannot read properties of undefined (reading 'map')
    at Object.render (/app/src/components/List.tsx:42:18)   ← Your code (first line with your path)
    at processChild (/app/node_modules/react-dom/...:1234)  ← Library code (ignore for source location)
    at ReactRoot.render (/app/node_modules/react-dom/...:567)
```

**Extraction rules:**
1. The **error type and message** (first line) → determines error category
2. The **first line with your file path** → exact location of the problem
3. **Ignore node_modules lines** → library internals, not your bug
4. The **call chain** (reading bottom to top) → shows how your code was called

### Reading a Python Traceback

```
Traceback (most recent call last):
  File "/app/src/main.py", line 12, in handler     ← Entry point (bottom of call stack)
  File "/app/src/service.py", line 45, in get_data ← Intermediate call
    result = self.client.fetch(url)
  File "/app/src/client.py", line 23, in fetch     ← Where error actually occurred
    return self.session.get(url).json()
AttributeError: 'NoneType' object has no attribute 'get'  ← The actual error
```

**Extraction rules:**
1. Read from **bottom to top** — the error is at the bottom
2. The **last file listed that's in your source** → where the fix goes
3. The **error type** (AttributeError) → determines category
4. The **specific line** in your code → look at what's `None` on that line

### Extracting Search Terms from a Stack Trace

```
From: AttributeError: 'NoneType' object has no attribute 'get'
at: /app/src/client.py:23 in fetch → self.session.get(url)

→ self.session is None
→ Search: "requests session None AttributeError python"
→ Or: "httpx session None" (check which HTTP client is used)
→ More specific: "requests.Session is None after {context}"
```
