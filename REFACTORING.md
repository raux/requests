# Refactoring Opportunities

This document catalogs refactoring opportunities identified across the `requests` library
source code, organized by refactoring type. Each entry references the specific file,
method, and line range where the opportunity exists.

---

## Table of Contents

- [1. Extract Method](#1-extract-method)
- [2. Replace Magic Numbers with Named Constants](#2-replace-magic-numbers-with-named-constants)
- [3. Introduce Parameter Object](#3-introduce-parameter-object)
- [4. Simplify Conditional Expressions](#4-simplify-conditional-expressions)
- [5. Remove Duplicated Code](#5-remove-duplicated-code)
- [6. Extract Class / Move Method](#6-extract-class--move-method)
- [7. Remove Dead Code](#7-remove-dead-code)
- [8. Replace Exception Handling with Dispatch](#8-replace-exception-handling-with-dispatch)
- [9. Address Technical Debt](#9-address-technical-debt)

---

## 1. Extract Method

Long methods that perform multiple responsibilities can be broken into smaller, focused
methods to improve readability and testability.

### 1.1 `sessions.py` — `resolve_redirects()` (Lines 160–282, 123 lines)

**Current state:** A single generator method that handles URL extraction, fragment
management, response history tracking, redirect limit validation, URL normalization,
header manipulation, cookie management, auth/proxy rebuilding, body rewind logic, and
request re-sending.

**Suggested extractions:**

- `_normalize_redirect_url(url, resp_url, previous_fragment)` — handle scheme-less
  URLs, fragment merging, and relative URL resolution (Lines 199–220)
- `_clean_headers_for_redirect(prepared_request, status_code)` — remove headers that
  should not be forwarded on redirect (Lines 225–233)
- `_prepare_cookies_for_redirect(prepared_request, req, resp)` — extract, merge, and
  prepare cookies (Lines 241–243)

### 1.2 `models.py` — `prepare_body()` (Lines 496–573, 78 lines)

**Current state:** Handles JSON encoding, stream detection, file position tracking,
multipart file encoding, and header management in a single method.

**Suggested extractions:**

- `_handle_json_body(data)` — JSON serialization and content-type setup
- `_is_stream_body(data)` — stream detection logic
- `_handle_stream_body(data)` — file position tracking and chunked encoding
- `_handle_form_body(data, files)` — multipart encoding and parameter handling

### 1.3 `models.py` — `prepare_url()` (Lines 411–484, 74 lines)

**Current state:** Performs URL normalization, host validation/IDNA encoding, netloc
building, and parameter encoding.

**Suggested extractions:**

- `_normalize_url(url)` — initial URL cleanup and scheme handling
- `_validate_and_encode_host(host)` — IDNA encoding and host validation
- `_build_netloc(parsed)` — construct the netloc component from parsed URL parts
- `_encode_url_params(url, params)` — append query parameters

### 1.4 `models.py` — `_encode_files()` (Lines 139–208, 70 lines)

**Current state:** Validates input, normalizes format, encodes fields, unpacks tuples
(2/3/4-tuple variants), handles file objects, creates RequestFields, and encodes
multipart data.

**Suggested extractions:**

- `_validate_files_input(files, data)` — input validation
- `_encode_form_fields(data)` — form field encoding
- `_unpack_file_data(file_entry)` — normalize 2/3/4-tuple file entries

### 1.5 `adapters.py` — `send()` (Lines 592–698, 107 lines)

**Current state:** Sets up connections, configures timeouts, executes requests, and
catches 9+ exception types.

**Suggested extractions:**

- `_prepare_timeout(timeout)` — timeout configuration (Lines 631–643)
- `_handle_request_exceptions(error, request, timeout)` — exception translation
  (Lines 660–696)
- `_setup_connection(request, proxies)` — connection setup (Lines 612–627)

### 1.6 `auth.py` — `build_digest_header()` (Lines 126–234, 109 lines)

**Current state:** Handles algorithm selection, hash function creation, hash value
computation, response digest building, and header construction in a single method.

**Suggested extractions:**

- `_get_hash_function(algorithm)` — algorithm selection and hash function creation
  (Lines 138–175)
- `_compute_digest_hashes(algorithm, ...)` — hash value computation (Lines 189–207)
- `_build_response_digest(...)` — response digest building (Lines 209–216)
- `_build_authorization_header(...)` — header construction (Lines 221–234)

### 1.7 `sessions.py` — `send()` (Lines 676–752, 77 lines)

**Current state:** Manages request defaults, proxy resolution, type validation, adapter
selection, request dispatch, hook processing, cookie persistence, and redirect handling.

**Suggested extractions:**

- `_apply_request_defaults(kwargs)` — setting default kwargs (Lines 683–687)
- `_dispatch_request(request, kwargs)` — send and time the request (Lines 700–710)
- `_handle_response_history(response, request, kwargs)` — redirect history
  (Lines 715–737)

### 1.8 `adapters.py` — `cert_verify()` (Lines 282–337, 56 lines)

**Current state:** Mixes server certificate verification with client certificate setup.

**Suggested extractions:**

- `_verify_server_cert(conn, url, verify)` — server cert verification (Lines 294–319)
- `_setup_client_cert(conn, cert)` — client cert configuration (Lines 321–336)

### 1.9 `models.py` — `iter_content()` (Lines 801–858, 58 lines)

**Current state:** Validates input, creates a stream generator with exception
translation, manages content caching, and handles unicode decoding.

**Suggested extractions:**

- `_validate_chunk_size(chunk_size)` — input validation
- `_create_stream_generator(chunk_size)` — raw stream generator
- `_select_content_source()` — choose between stream and cached content

### 1.10 `auth.py` — `handle_401()` (Lines 241–283, 43 lines)

**Current state:** Handles status validation, request retry logic, and digest
processing with 4-level deep nesting.

**Suggested extractions:**

- `_should_handle_401(response)` — guard clause for status validation
- `_prepare_retry_request(response)` — prepare the retry request
- `_send_retry_request(response, request, kwargs)` — send and process retry

---

## 2. Replace Magic Numbers with Named Constants

Hardcoded numeric and string literals reduce readability and make maintenance harder.

### 2.1 `models.py` — HTTP Status Code Ranges

| Line(s) | Value | Suggested Constant |
|---------|-------|--------------------|
| 1017, 1022 | `400` | `HTTP_CLIENT_ERROR_MIN` |
| 1022 | `500` | `HTTP_SERVER_ERROR_MIN` |
| 1022 | `600` | `HTTP_STATUS_MAX` |

### 2.2 `models.py` — Encoding Defaults

| Line(s) | Value | Suggested Constant |
|---------|-------|--------------------|
| 130, 131, 168, 171, 406, 517, 1011 | `"utf-8"` | `DEFAULT_ENCODING` |
| 1013 | `"iso-8859-1"` | `FALLBACK_ENCODING` |

### 2.3 `models.py` — File Tuple Sizes

| Line(s) | Value | Suggested Constant |
|---------|-------|--------------------|
| 180 | `2` | `FILE_TUPLE_BASIC` |
| 182 | `3` | `FILE_TUPLE_WITH_CONTENT_TYPE` |
| 185 | `4` | `FILE_TUPLE_WITH_HEADERS` |

### 2.4 `sessions.py` — Port Numbers and Headers

| Line(s) | Value | Suggested Constant |
|---------|-------|--------------------|
| 140 | `80` | `DEFAULT_HTTP_PORT` |
| 142 | `443` | `DEFAULT_HTTPS_PORT` |
| 230 | `("Content-Length", "Content-Type", "Transfer-Encoding")` | `REDIRECT_STRIP_HEADERS` |

### 2.5 `auth.py` — Digest Authentication

| Line(s) | Value | Suggested Constant |
|---------|-------|--------------------|
| 199 | `"08x"` | `NONCE_COUNT_FORMAT` |
| 205 | `[:16]` | `CNONCE_LENGTH` |
| 260 | `< 2` | `MAX_DIGEST_AUTH_RETRIES` |

### 2.6 `utils.py` — Network and BOM Constants

| Line(s) | Value | Suggested Constant |
|---------|-------|--------------------|
| 692 | `0xFFFFFFFF` | `IPV4_NETMASK_MAX` |
| 692, 719 | `32` | `IPV4_BITS` |
| 943 | `[:4]` | `UTF_BOM_SAMPLE_SIZE` |

---

## 3. Introduce Parameter Object

Methods with long parameter lists can be simplified by grouping related parameters into
an object.

### 3.1 `sessions.py` — `request()` (16 parameters)

```python
def request(self, method, url, params, data, headers, cookies,
            files, auth, timeout, allow_redirects, proxies,
            hooks, stream, verify, cert, json):
```

**Suggestion:** Group transport-related parameters (`timeout`, `allow_redirects`,
`proxies`, `stream`, `verify`, `cert`) into a `RequestOptions` object.

### 3.2 `models.py` — `Request.__init__()` / `PreparedRequest.prepare()` (10 parameters)

```python
def prepare(self, method, url, headers, files, data,
            params, auth, cookies, hooks, json):
```

**Suggestion:** Group content-related parameters (`data`, `files`, `json`) and
auth/cookie parameters into dedicated objects.

### 3.3 `sessions.py` — `resolve_redirects()` (9 parameters)

```python
def resolve_redirects(self, resp, req, stream, timeout,
                      verify, cert, proxies, yield_requests, **adapter_kwargs):
```

**Suggestion:** Group transport parameters (`stream`, `timeout`, `verify`, `cert`,
`proxies`) into a `RedirectOptions` object.

---

## 4. Simplify Conditional Expressions

Deeply nested conditionals and complex branching logic can be simplified using guard
clauses, lookup tables, or helper methods.

### 4.1 `auth.py` — Algorithm Selection Chain (Lines 138–174)

**Current state:** A 4-way `if`/`elif` chain to select hash algorithms.

**Suggestion:** Replace with a dictionary mapping:

```python
HASH_ALGORITHMS = {
    "MD5": hashlib.md5,
    "MD5-SESS": hashlib.md5,
    "SHA": hashlib.sha1,
    "SHA-256": hashlib.sha256,
    "SHA-512": hashlib.sha512,
}
```

### 4.2 `sessions.py` — `rebuild_method()` (Lines 334–356)

**Current state:** Three separate `if` statements checking `status_code`:

```python
if response.status_code == codes.see_other and method != "HEAD":
    method = "GET"
if response.status_code == codes.found and method != "HEAD":
    method = "GET"
if response.status_code == codes.moved and method == "POST":
    method = "GET"
```

**Suggestion:** Replace with a decision table mapping status codes to method rewrite
rules.

### 4.3 `cookies.py` — Nested Cookie Filtering (Lines 378–382, 399–409)

**Current state:** Three levels of nested `if` checks for name, domain, and path.

**Suggestion:** Extract a `_matches_criteria(cookie, name, domain, path)` predicate
method.

### 4.4 `utils.py` — `guess_json_utf()` (Lines 945–974)

**Current state:** 8 return paths with 3-level nesting and magic number comparisons.

**Suggestion:** Extract `_detect_bom()` and `_detect_by_null_pattern()` helper methods.

### 4.5 `sessions.py` — `should_strip_auth()` (Lines 128–159)

**Current state:** Multiple conditions checking port numbers with duplicated logic.

**Suggestion:** Extract `_is_http_to_https_upgrade(old_parsed, new_parsed)` helper.

---

## 5. Remove Duplicated Code

Identical or near-identical code patterns that appear in multiple locations.

### 5.1 `models.py` — UTF-8 Encoding Ternary (6+ occurrences)

**Pattern:**

```python
k.encode("utf-8") if isinstance(k, str) else k
```

**Suggestion:** Extract a `_encode_if_str(value, encoding="utf-8")` utility function.

### 5.2 `models.py` — Type Checking Patterns (23+ occurrences)

**Patterns:**

```python
isinstance(x, (str, bytes, basestring))
hasattr(x, "__iter__")
hasattr(x, "read")
```

**Suggestion:** Create `_is_string_like()`, `_is_iterable()`, `_is_file_like()` helpers.

### 5.3 `models.py` — Key-Value Encoding Loop (Lines 121–133 and 157–172)

**Current state:** Both `_encode_params()` and `_encode_files()` iterate over key-value
pairs, convert to lists, and encode values with near-identical logic.

**Suggestion:** Extract a common `_encode_key_value_list(data)` helper.

### 5.4 `models.py` — Decode with Fallback (Lines 967–973 and 1005–1015)

**Pattern:**

```python
try:
    result = data.decode("utf-8")
except UnicodeDecodeError:
    result = data.decode("iso-8859-1")
```

**Suggestion:** Extract a `_safe_decode(data, encoding, fallback)` helper.

### 5.5 `auth.py` — Hash Function Factory (Lines 145–172)

**Current state:** Four near-identical functions (`md5_utf8`, `sha_utf8`,
`sha256_utf8`, `sha512_utf8`) that only differ in the hash algorithm.

**Suggestion:** Create a factory function:

```python
def _make_hash_utf8(hash_func):
    def hasher(x):
        if isinstance(x, str):
            x = x.encode("utf-8")
        return hash_func(x).hexdigest()
    return hasher
```

### 5.6 `auth.py` — Type Validation for Credentials (Lines 35–53)

**Current state:** Identical validation logic repeated for both `username` and
`password`.

**Suggestion:** Extract `_ensure_string_credential(value, param_name)`.

### 5.7 `auth.py` — `HTTPBasicAuth` vs `HTTPProxyAuth` (Lines 76–104)

**Current state:** Nearly identical classes that only differ in which header they set
(`Authorization` vs `Proxy-Authorization`).

**Suggestion:** Parameterize the base class with a `header_name` attribute.

### 5.8 `cookies.py` — `list_domains()` vs `list_paths()` (Lines 277–291)

**Current state:** Identical iteration logic over cookies, differing only in the
attribute accessed.

**Suggestion:** Extract `_extract_unique_cookie_attr(attribute_name)`.

### 5.9 `cookies.py` — `_find()` vs `_find_no_duplicates()` (Lines 366–413)

**Current state:** Both methods use identical 3-level nested filtering logic.

**Suggestion:** Extract `_find_matching_cookies(name, domain, path)` generator.

### 5.10 `utils.py` — Type Validation (Lines 327 and 353)

**Current state:** Identical `isinstance` + `ValueError` check in two functions.

**Suggestion:** Extract `_validate_not_primitive(value)`.

### 5.11 `utils.py` — CIDR Parsing (Lines 679, 715, 723)

**Current state:** Same `split("/")` pattern repeated three times.

**Suggestion:** Extract `_parse_cidr(cidr_string)`.

### 5.12 `utils.py` — Deprecation Warnings (Lines 482–487 and 591–597)

**Current state:** Nearly identical warning text in two locations.

**Suggestion:** Extract `_emit_deprecation_warning(func_name, message)`.

### 5.13 `adapters.py` — Proxy URL Validation (Lines 457–461 and 500–504)

**Current state:** Identical proxy URL host validation in both
`get_connection_with_tls_context()` and the deprecated `get_connection()`.

**Suggestion:** Extract `_validate_proxy_url(proxy_url)`.

---

## 6. Extract Class / Move Method

Groups of related functionality that could be encapsulated in their own class.

### 6.1 `sessions.py` — Redirect Handling

**Current state:** `resolve_redirects()`, `rebuild_auth()`, `rebuild_proxies()`,
`rebuild_method()`, and `should_strip_auth()` are all redirect-related methods inside
the `Session` class.

**Suggestion:** Extract a `RedirectHandler` class to encapsulate redirect logic,
reducing `Session` class complexity.

### 6.2 `models.py` — Body Preparation

**Current state:** `prepare_body()`, `_encode_params()`, and `_encode_files()` in
`PreparedRequest` handle all body encoding concerns.

**Suggestion:** Extract a `BodyEncoder` class responsible for all body serialization
strategies.

### 6.3 `cookies.py` — Cookie Attribute Extraction

**Current state:** `list_domains()`, `list_paths()`, `multiple_domains()`, `_find()`,
and `_find_no_duplicates()` all iterate over cookies with similar patterns.

**Suggestion:** Consolidate filtering/iteration into a dedicated query interface.

---

## 7. Remove Dead Code

Code that is unreachable, unused, or produces no effect.

### 7.1 `cookies.py` — `MockResponse.getheaders()` Missing Return (Line 120–121)

```python
def getheaders(self, name):
    self._headers.getheaders(name)  # Missing return statement
```

**Action:** Add `return` or remove if unused.

### 7.2 `models.py` — Unused `self._next` Attribute (Line 663)

**Current state:** `self._next` is initialized in `Response.__init__()` but never
assigned after initialization.

**Action:** Verify usage and remove if unnecessary.

### 7.3 `utils.py` — Unused `tried_encodings` List (Line 600)

**Current state:** In `get_unicode_from_response()`, the `tried_encodings` list is
created and appended to but never read.

**Action:** Remove the unused variable.

### 7.4 `adapters.py` — No-Op Timeout Branch (Lines 640–641)

```python
elif isinstance(timeout, TimeoutSauce):
    pass
```

**Action:** Remove the branch or combine with the `else` clause.

---

## 8. Replace Exception Handling with Dispatch

Complex exception translation chains can be simplified using mapping tables.

### 8.1 `adapters.py` — Exception Translation Chain (Lines 660–696)

**Current state:** 9+ exception types handled with nested `isinstance` checks:

```python
except MaxRetryError as e:
    if isinstance(e.reason, ConnectTimeoutError):
        if not isinstance(e.reason, NewConnectionError):
            raise ConnectTimeout(...)
    if isinstance(e.reason, ResponseError):
        raise RetryError(...)
    if isinstance(e.reason, _ProxyError):
        raise ProxyError(...)
    ...
```

**Suggestion:** Replace with a dispatch table mapping exception types to handler
functions or replacement exceptions.

---

## 9. Address Technical Debt

Existing TODO, FIXME, and XXX comments indicating known incomplete implementations.

| File | Line(s) | Comment | Action |
|------|---------|---------|--------|
| `hooks.py` | 20 | `# TODO: response is the only one` | Evaluate if hook system needs expansion or simplification |
| `adapters.py` | 665 | `# TODO: Remove this in 3.0.0: see #2811` | Track for v3.0 release |
| `auth.py` | 181 | `# XXX not implemented yet` (auth-int qop) | Implement or raise `NotImplementedError` |
| `auth.py` | 215 | `# XXX handle auth-int.` | Implement or raise `NotImplementedError` |
| `auth.py` | 220 | `# XXX should the partial digests be encoded too?` | Research and resolve |
