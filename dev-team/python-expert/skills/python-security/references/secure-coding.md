# Secure Coding Practices for Python

Comprehensive reference covering path traversal, command injection, SSRF prevention, SQL injection, template injection, XML security, timing attacks, file upload validation, and logging security.

---

## Path Traversal Prevention

### Using pathlib (Recommended)

```python
from pathlib import Path

UPLOAD_DIR = Path("/app/uploads").resolve()

def safe_file_path(user_filename: str) -> Path:
    """Resolve a user-provided filename to a safe absolute path."""
    # Remove any directory components
    clean_name = Path(user_filename).name  # strips all directory parts

    # Resolve and verify
    full_path = (UPLOAD_DIR / clean_name).resolve()

    if not full_path.is_relative_to(UPLOAD_DIR):
        raise ValueError("Path traversal detected")

    return full_path
```

### Using os.path (Legacy)

```python
import os

UPLOAD_DIR = os.path.realpath("/app/uploads")

def safe_file_path_legacy(user_filename: str) -> str:
    # os.path.basename strips directory components
    clean_name = os.path.basename(user_filename)
    full_path = os.path.realpath(os.path.join(UPLOAD_DIR, clean_name))

    if not full_path.startswith(UPLOAD_DIR + os.sep):
        raise ValueError("Path traversal detected")

    return full_path
```

---

## Command Injection Prevention

### Secure Subprocess Calls

```python
import subprocess

# SAFE — argument list, no shell interpretation
def get_file_info(filepath: str) -> str:
    result = subprocess.run(
        ["file", "--mime-type", filepath],
        capture_output=True,
        text=True,
        timeout=10,
        check=True,
    )
    return result.stdout.strip()

# SAFE — using shlex.quote when shell is unavoidable
import shlex

def grep_file(pattern: str, filepath: str) -> str:
    # Still prefer argument list form over this
    safe_pattern = shlex.quote(pattern)
    safe_filepath = shlex.quote(filepath)
    result = subprocess.run(
        ["grep", "-n", pattern, filepath],
        capture_output=True,
        text=True,
        timeout=10,
    )
    return result.stdout
```

---

## SSRF Prevention

Server-Side Request Forgery allows attackers to make requests to internal services:

```python
import ipaddress
from urllib.parse import urlparse

BLOCKED_NETWORKS = [
    ipaddress.ip_network("127.0.0.0/8"),      # localhost
    ipaddress.ip_network("10.0.0.0/8"),        # private
    ipaddress.ip_network("172.16.0.0/12"),     # private
    ipaddress.ip_network("192.168.0.0/16"),    # private
    ipaddress.ip_network("169.254.0.0/16"),    # link-local
    ipaddress.ip_network("::1/128"),           # IPv6 localhost
    ipaddress.ip_network("fc00::/7"),          # IPv6 private
]

def validate_url(url: str, allowed_schemes: set[str] | None = None) -> str:
    """Validate a URL is safe to fetch (no SSRF)."""
    if allowed_schemes is None:
        allowed_schemes = {"https"}

    parsed = urlparse(url)

    # Check scheme
    if parsed.scheme not in allowed_schemes:
        raise ValueError(f"URL scheme '{parsed.scheme}' not allowed")

    # Check hostname
    if not parsed.hostname:
        raise ValueError("No hostname in URL")

    # Resolve DNS and check IP
    import socket
    try:
        resolved_ips = socket.getaddrinfo(parsed.hostname, parsed.port or 443)
    except socket.gaierror:
        raise ValueError("Could not resolve hostname")

    for family, _, _, _, addr in resolved_ips:
        ip = ipaddress.ip_address(addr[0])
        for network in BLOCKED_NETWORKS:
            if ip in network:
                raise ValueError(f"URL resolves to blocked network: {network}")

    return url
```

---

## SQL Injection Prevention

### Parameterized Queries

```python
import sqlite3

# DANGEROUS — string formatting
def get_user_unsafe(email: str):
    conn.execute(f"SELECT * FROM users WHERE email = '{email}'")

# SAFE — parameterized query
def get_user_safe(email: str):
    conn.execute("SELECT * FROM users WHERE email = ?", (email,))

# SAFE with named parameters
def search_products(name: str, min_price: float):
    conn.execute(
        "SELECT * FROM products WHERE name LIKE :name AND price >= :min_price",
        {"name": f"%{name}%", "min_price": min_price},
    )
```

### SQLAlchemy

```python
from sqlalchemy import select, text

# SAFE — ORM query (always parameterized)
stmt = select(User).where(User.email == email)

# SAFE — text() with bound parameters
stmt = text("SELECT * FROM users WHERE email = :email")
result = session.execute(stmt, {"email": email})

# DANGEROUS — f-string with text()
stmt = text(f"SELECT * FROM users WHERE email = '{email}'")
```

---

## Template Injection

### Jinja2 Sandboxed Environment

```python
from jinja2 import SandboxedEnvironment, select_autoescape

# SAFE — sandboxed environment restricts dangerous operations
env = SandboxedEnvironment(
    autoescape=select_autoescape(["html", "xml"]),
)

# Renders with HTML escaping, blocks dangerous operations
template = env.from_string("Hello, {{ name }}!")
result = template.render(name=user_input)

# DANGEROUS — rendering user input as a template
from jinja2 import Environment
env = Environment()
template = env.from_string(user_controlled_template)  # RCE risk!
```

### Never Use eval/exec with User Input

```python
# DANGEROUS — arbitrary code execution
eval(user_input)
exec(user_input)
compile(user_input, "<string>", "exec")

# SAFE — use ast.literal_eval for simple data structures
import ast
data = ast.literal_eval("[1, 2, 3]")  # only evaluates literals
```

---

## XML Security

### Preventing XXE (XML External Entity) Attacks

```python
# DANGEROUS — standard library XML parsers are vulnerable to XXE
import xml.etree.ElementTree as ET
tree = ET.parse(untrusted_xml_file)  # can read local files, make network requests

# SAFE — use defusedxml
# pip install defusedxml
import defusedxml.ElementTree as ET
tree = ET.parse(untrusted_xml_file)  # XXE attacks blocked

# defusedxml also provides safe versions of:
from defusedxml import minidom, sax, pulldom, expatreader
from defusedxml.common import EntitiesForbidden

try:
    tree = ET.parse(untrusted_xml)
except EntitiesForbidden:
    print("XML contained forbidden entities")
```

---

## Timing Attacks

### Constant-Time Comparison

```python
import hmac
import secrets

# DANGEROUS — early return leaks information about which characters match
def insecure_compare(a: str, b: str) -> bool:
    if len(a) != len(b):
        return False
    for x, y in zip(a, b):
        if x != y:
            return False  # returns faster for early mismatches
    return True

# SAFE — constant-time comparison
def secure_compare(a: str, b: str) -> bool:
    return hmac.compare_digest(a.encode(), b.encode())

# Also available via secrets module
secrets.compare_digest(a, b)
```

---

## File Upload Validation

```python
import magic  # pip install python-magic
from pathlib import Path

ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".gif", ".pdf"}
ALLOWED_MIME_TYPES = {
    "image/jpeg", "image/png", "image/gif", "application/pdf"
}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB

def validate_upload(file_data: bytes, filename: str) -> Path:
    """Validate an uploaded file for security."""
    # 1. Check file size
    if len(file_data) > MAX_FILE_SIZE:
        raise ValueError(f"File exceeds {MAX_FILE_SIZE} byte limit")

    # 2. Check extension
    ext = Path(filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise ValueError(f"Extension '{ext}' not allowed")

    # 3. Check MIME type from file content (not the declared type)
    mime_type = magic.from_buffer(file_data, mime=True)
    if mime_type not in ALLOWED_MIME_TYPES:
        raise ValueError(f"MIME type '{mime_type}' not allowed")

    # 4. Generate safe storage name
    import uuid
    safe_name = f"{uuid.uuid4()}{ext}"
    return Path(safe_name)
```

---

## Logging Security

### Never Log Secrets

```python
import logging

logger = logging.getLogger(__name__)

# DANGEROUS
logger.info(f"User authenticated with token: {token}")
logger.debug(f"Database connection: {connection_string}")

# SAFE — log identifiers, not secrets
logger.info("User %s authenticated successfully", user_id)
logger.debug("Connected to database %s", db_name)
```

### PII Masking in Logs

```python
import re

def mask_pii(message: str) -> str:
    """Mask common PII patterns in log messages."""
    # Email addresses
    message = re.sub(r'\b[\w.+-]+@[\w-]+\.[\w.-]+\b', '<EMAIL>', message)
    # Credit card numbers
    message = re.sub(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b', '<CREDIT_CARD>', message)
    # SSN
    message = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '<SSN>', message)
    # Phone numbers
    message = re.sub(r'\b\+?1?[\s-]?\(?\d{3}\)?[\s-]?\d{3}[\s-]?\d{4}\b', '<PHONE>', message)
    return message

class PIIMaskingFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.msg = mask_pii(str(record.msg))
        return True

logger.addFilter(PIIMaskingFilter())
```
