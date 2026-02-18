# Error Analysis Reference

Systematic error taxonomy, diagnostic checklists, and canonical search queries by error category.

---

## Error Taxonomy

### JavaScript / TypeScript Runtime Errors

#### TypeError

Caused by: Operating on a value of the wrong type — usually `undefined` or `null` where an object is expected.

```
TypeError: Cannot read properties of undefined (reading 'X')
TypeError: X is not a function
TypeError: Cannot set properties of null (setting 'X')
TypeError: X is not iterable
```

**Diagnostic questions:**
1. What is the variable that is `undefined`? (Look at the line number in the stack trace)
2. Should this variable have been initialized? If yes, why wasn't it?
3. Is this asynchronous data that hasn't loaded yet?
4. Was the variable returned from a function that may return `undefined`?

**Canonical search queries:**
```
"Cannot read properties of undefined" {property-name} react/node/express
"X is not a function" {context/library}
"Cannot read properties of null" {property} fix
```

#### ReferenceError

Caused by: Accessing a variable that doesn't exist in scope.

```
ReferenceError: X is not defined
ReferenceError: Cannot access 'X' before initialization (TDZ)
```

**Diagnostic questions:**
1. Is there a missing import statement?
2. Is there a typo in the variable name?
3. Is the variable declared with `let`/`const` but accessed before the declaration line?
4. Is the variable in a different scope than assumed?

**Canonical search queries:**
```
"X is not defined" {bundler/framework}
"Cannot access before initialization" {variable-type}
ReferenceError {library} module scope fix
```

#### SyntaxError

Caused by: Code that cannot be parsed.

```
SyntaxError: Unexpected token 'X'
SyntaxError: Unexpected end of JSON input
SyntaxError: Cannot use import statement outside a module
```

**Diagnostic questions:**
1. Is this a JSON parse error? (Check the raw JSON for unterminated strings, trailing commas)
2. Is `import` used in a CommonJS context? (Check `package.json` for `"type"` field)
3. Is a Babel/TypeScript transform missing or misconfigured?
4. Is there a stray character from a bad merge conflict?

**Canonical search queries:**
```
"Unexpected token" {context} fix
"Cannot use import statement outside a module" node {version}
"Unexpected end of JSON input" {library}
```

---

### Node.js / File System Errors

#### ENOENT (No Such File or Directory)

```
Error: ENOENT: no such file or directory, open '/path/to/file'
Error: Cannot find module './path/to/module'
```

**Diagnostic checklist:**
- [ ] Is the path relative or absolute? (Relative paths depend on `process.cwd()`, not the file location)
- [ ] Does the file actually exist? (`ls -la /path/to/file`)
- [ ] Is there a typo in the path (case-sensitive on Linux)?
- [ ] For module not found: Is the package installed? (`node_modules/{package}` exists?)
- [ ] Is the working directory what you think it is? (`console.log(process.cwd())`)

**Canonical search queries:**
```
"ENOENT" {operation} node path fix
"Cannot find module" {module-name} node
```

#### EACCES (Permission Denied)

```
Error: EACCES: permission denied, open '/path/to/file'
npm error: EACCES permission denied access '/usr/local/lib'
```

**Diagnostic checklist:**
- [ ] What user is the process running as?
- [ ] What are the file permissions? (`ls -la /path`)
- [ ] For npm: was npm installed with sudo? (Root-owned global installs cause this)
- [ ] For containers: does the running user have volume mount permissions?

**Canonical search queries:**
```
"EACCES permission denied" npm fix
"permission denied" {operation} node linux fix
```

#### ECONNREFUSED / ETIMEDOUT

```
Error: connect ECONNREFUSED 127.0.0.1:5432
Error: connect ETIMEDOUT {ip}:{port}
```

**Diagnostic checklist:**
- [ ] Is the target service running? (`ps aux | grep {service}` or `docker ps`)
- [ ] Is the port correct? (Check config vs actual service port)
- [ ] Is the host correct? (localhost vs 0.0.0.0 vs container name in Docker)
- [ ] Is there a firewall or security group blocking the port?
- [ ] For ETIMEDOUT: is there a proxy or VPN that might be intercepting?

**Canonical search queries:**
```
"ECONNREFUSED" {database/service} docker fix
"ETIMEDOUT" {service} connection fix
```

---

### Python Errors

#### ImportError / ModuleNotFoundError

```
ImportError: cannot import name 'X' from 'Y'
ModuleNotFoundError: No module named 'X'
```

**Diagnostic checklist:**
- [ ] Is the package installed in the active virtual environment?
  ```bash
  pip list | grep {package}
  which python  # Confirms which env is active
  ```
- [ ] Is the module name spelled correctly? (Case-sensitive)
- [ ] For `cannot import name`: was the name renamed in a newer version?
  ```bash
  python -c "import {package}; print(dir({package}))"
  ```
- [ ] Is there a circular import? (A imports B, B imports A)
- [ ] Is `__init__.py` missing from a local package directory?

**Canonical search queries:**
```
"No module named" {module} python {version} virtualenv fix
"cannot import name" {name} {library} {version}
```

#### AttributeError

```
AttributeError: 'X' object has no attribute 'Y'
AttributeError: module 'X' has no attribute 'Y'
```

**Diagnostic checklist:**
- [ ] Is this a version mismatch? (Attribute may have been renamed or removed)
  ```bash
  python -c "import {module}; print(dir({module}.{class}))"
  ```
- [ ] Is the object `None` where an object is expected?
- [ ] Is there a typo in the attribute name?
- [ ] Was the import done correctly? (`from x import Y` vs `import x; x.Y`)

**Canonical search queries:**
```
"{module} object has no attribute '{attr}'" {version} fix
"AttributeError" {library} {class} {version}
```

---

### HTTP / Network Errors

#### 401 Unauthorized

**Diagnostic checklist:**
- [ ] Is the Authorization header present in the request?
- [ ] Is the token expired? (JWT: decode and check `exp` claim)
- [ ] Is the token format correct? (`Bearer {token}` vs just `{token}`)
- [ ] Is the token being sent to the right domain/endpoint?

#### 403 Forbidden

**Diagnostic checklist:**
- [ ] Is the user authenticated but lacks permission?
- [ ] Is there a CORS issue? (Browser blocks due to missing CORS headers — check Network tab)
- [ ] Is there an IP allowlist blocking the request?
- [ ] For storage (S3, GCS): are bucket permissions correctly configured?

#### 500 Internal Server Error

**Diagnostic checklist:**
- [ ] Check the server logs — the real error is always server-side
- [ ] Is there an unhandled exception in the route handler?
- [ ] Is this a database connection failure?
- [ ] Is middleware throwing an unexpected error?

**Canonical search queries:**
```
{library/framework} 500 error logging debug
{service} "internal server error" {operation} fix
```

---

## Intermittent Error Checklist

For errors that don't always occur:

```
[ ] Race condition? (Multiple async operations with shared state)
[ ] Resource exhaustion? (Memory leak, connection pool exhausted, file descriptor limit)
[ ] Timing dependency? (Server not ready before client connects)
[ ] Environment-specific? (Works locally, fails in CI — check env vars, OS, dependencies)
[ ] Load-dependent? (Only fails under high concurrency)
[ ] Data-dependent? (Only fails with specific input shape)
```

**Search queries for intermittent errors:**
```
"{library}" flaky intermittent {error-type} race condition
"{error}" {library} only sometimes {environment}
```

---

## Version Conflict Checklist

When errors occur after a dependency update:

```
[ ] What changed? (diff package.json or requirements.txt against last working commit)
[ ] Is there a peer dependency warning? (npm install output shows peer dependency conflicts)
[ ] Does the changelog mention a breaking change for this version?
[ ] Is there a migration guide?
```

**Canonical search queries:**
```
{library} "breaking change" {version} migration
{library} peer dependency {version} fix
{library} upgrade v{old} to v{new} errors
```
