---
name: Python Security
description: This skill should be used when the user asks about "Python security", "input validation", "secrets management", "subprocess safety", "pickle security", "dependency audit", "SSRF prevention", "path traversal", "deserialization risk", "pip-audit", or "Python cryptography". It covers secure coding practices, secrets management, subprocess safety, deserialization risks, dependency auditing, and cryptographic patterns for Python applications.
---

## Input Validation and Sanitization

### Path Traversal Prevention

Never use string concatenation or `os.path.join` alone with user input:

```python
from pathlib import Path

SAFE_BASE = Path("/app/uploads").resolve()

def safe_read_file(user_path: str) -> str:
    """Read a file, preventing path traversal attacks."""
    requested = (SAFE_BASE / user_path).resolve()

    if not requested.is_relative_to(SAFE_BASE):
        raise ValueError("Path traversal detected")

    if not requested.is_file():
        raise FileNotFoundError("File not found")

    return requested.read_text()
```

### Pydantic Validation for User Input

```python
from pydantic import BaseModel, Field, field_validator
import re

class UserInput(BaseModel):
    username: str = Field(min_length=3, max_length=30)
    email: str = Field(max_length=255)
    query: str = Field(max_length=1000)

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_-]+$", v):
            raise ValueError("Username may only contain letters, digits, hyphens, and underscores")
        return v

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if not re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", v):
            raise ValueError("Invalid email format")
        return v.lower()
```

### ReDoS Prevention

Avoid catastrophic backtracking in regular expressions:

```python
# DANGEROUS — exponential backtracking on long inputs
import re
evil_pattern = re.compile(r"^(a+)+$")

# SAFE — use atomic grouping or rewrite the pattern
safe_pattern = re.compile(r"^a+$")

# For complex patterns, use the `re2` library (Google RE2 engine) which guarantees linear time
# pip install google-re2
import re2
pattern = re2.compile(r"^(a+)+$")  # safe — RE2 prevents backtracking
```

---

## Secrets Management

### The `secrets` Module

Always use `secrets` for security-sensitive randomness. Never use `random`:

```python
import secrets

# Generate a URL-safe token (e.g., password reset links)
token = secrets.token_urlsafe(32)  # 43-character base64 string

# Generate a hex token
api_key = secrets.token_hex(32)  # 64-character hex string

# Generate a random integer in range
otp = secrets.randbelow(1_000_000)  # 0 to 999999

# Compare strings in constant time (prevents timing attacks)
is_valid = secrets.compare_digest(user_token, stored_token)
```

### Environment Variable Handling

```python
import os
from pydantic_settings import BaseSettings

class AppSettings(BaseSettings):
    database_url: str
    secret_key: str
    api_key: str
    debug: bool = False

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = AppSettings()

# NEVER do this:
SECRET = "hardcoded-secret-value"
```

### .env File Handling

```bash
# .env (NEVER commit this file)
DATABASE_URL=postgres://user:password@localhost/db
SECRET_KEY=your-secret-key-here
API_KEY=sk-...
```

```gitignore
# .gitignore
.env
.env.*
!.env.example
```

---

## Subprocess Safety

### Shell=True Is Dangerous

```python
import subprocess

# DANGEROUS — shell injection
user_input = "file.txt; rm -rf /"
subprocess.run(f"cat {user_input}", shell=True)  # executes both commands

# SAFE — argument list, no shell interpretation
subprocess.run(["cat", user_input])  # "file.txt; rm -rf /" treated as a filename
```

### Secure Subprocess Pattern

```python
import subprocess
import shlex

def run_command(args: list[str], timeout: int = 30) -> subprocess.CompletedProcess:
    """Run a command securely with timeout and output capture."""
    return subprocess.run(
        args,
        capture_output=True,
        text=True,
        timeout=timeout,
        check=False,  # don't raise on non-zero exit
        shell=False,  # never use shell=True with user input
    )

# If you MUST construct a shell command string (rare), use shlex.quote:
filename = shlex.quote(user_input)
# But prefer argument lists over shell=True in all cases
```

---

## Deserialization Risks

### Pickle Is Unsafe

`pickle.load()` executes arbitrary code. Never unpickle untrusted data:

```python
import pickle

# DANGEROUS — arbitrary code execution
class Exploit:
    def __reduce__(self):
        import os
        return (os.system, ("rm -rf /",))

data = pickle.dumps(Exploit())
pickle.loads(data)  # executes os.system("rm -rf /")
```

### Safe Alternatives

```python
import json
import msgpack  # pip install msgpack

# JSON — safe, human-readable
data = json.loads(untrusted_json_string)

# msgpack — safe, efficient binary format
data = msgpack.unpackb(untrusted_bytes, raw=False)

# If you MUST use pickle, restrict classes with a custom Unpickler:
import io

class RestrictedUnpickler(pickle.Unpickler):
    ALLOWED_CLASSES = {
        ("builtins", "set"),
        ("builtins", "frozenset"),
        ("collections", "OrderedDict"),
    }

    def find_class(self, module: str, name: str):
        if (module, name) not in self.ALLOWED_CLASSES:
            raise pickle.UnpicklingError(f"Forbidden class: {module}.{name}")
        return super().find_class(module, name)

def safe_load(data: bytes):
    return RestrictedUnpickler(io.BytesIO(data)).load()
```

### YAML Safety

```python
import yaml

# DANGEROUS — yaml.load can execute arbitrary Python
data = yaml.load(untrusted_yaml, Loader=yaml.Loader)

# SAFE — yaml.safe_load only loads basic types
data = yaml.safe_load(untrusted_yaml)
```

---

## Dependency Security

### pip-audit

```bash
# Install
pip install pip-audit

# Audit current environment
pip-audit

# Audit a requirements file
pip-audit -r requirements.txt

# Output as JSON (for CI)
pip-audit --format json --output audit-results.json

# Fix vulnerable packages
pip-audit --fix
```

### Safety

```bash
# Install
pip install safety

# Check current environment
safety check

# Check a requirements file
safety check -r requirements.txt
```

### CI Integration (GitHub Actions)

```yaml
- name: Audit dependencies
  run: |
    pip install pip-audit
    pip-audit --format json --output audit.json || true
    pip-audit --desc

- name: Upload audit results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: dependency-audit
    path: audit.json
```

---

## Cryptography Basics

### Password Hashing

Never use MD5, SHA1, or SHA256 for passwords. Use bcrypt or argon2:

```python
# bcrypt
import bcrypt

def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12)).decode()

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode(), hashed.encode())

# argon2 (recommended)
from argon2 import PasswordHasher

ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)

hashed = ph.hash(password)
try:
    ph.verify(hashed, password)
except Exception:
    print("Invalid password")
```

### HMAC for Message Authentication

```python
import hmac
import hashlib

def sign_message(message: str, secret: str) -> str:
    return hmac.new(secret.encode(), message.encode(), hashlib.sha256).hexdigest()

def verify_signature(message: str, signature: str, secret: str) -> bool:
    expected = sign_message(message, secret)
    return hmac.compare_digest(signature, expected)  # constant-time comparison
```

### Token Generation

```python
import secrets

# URL-safe token for password reset links, email verification
reset_token = secrets.token_urlsafe(32)

# Hex token for API keys
api_key = secrets.token_hex(32)

# Numeric OTP
otp = f"{secrets.randbelow(1_000_000):06d}"  # zero-padded 6 digits
```

---

## References

- **[Secure Coding Practices](references/secure-coding.md)** — Path traversal, command injection, SSRF prevention, SQL injection, template injection, XML security, timing attacks, file upload validation, and logging security.
- **[Dependency Management](references/dependency-management.md)** — pip-audit, safety, CVE analysis, lock file strategies, automated updates, supply chain attack patterns, private indexes, and reproducible builds.
