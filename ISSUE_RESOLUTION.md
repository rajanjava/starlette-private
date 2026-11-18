# Issue #3786: Empty Elements in `Access-Control-Request-Headers` (CORS)

## 📌 Overview & Metadata
* **Upstream Project:** [encode/starlette](https://github.com/encode/starlette)
* **Upstream Issue:** [Issue #3786](https://github.com/encode/starlette/issues/3786)
* **Status:** ✅ Resolved & Verified
* **Component:** `starlette.middleware.cors.CORSMiddleware`
* **Affected File:** `starlette/middleware/cors.py`
* **Test File:** `tests/middleware/test_cors.py`

---

## 🔍 Problem Description

In `CORSMiddleware`, preflight (`OPTIONS`) requests are evaluated against configured CORS policies. If a browser or HTTP proxy sends preflight requests where `Access-Control-Request-Headers` contains empty elements (such as extra spaces, double commas, or trailing commas), Starlette rejected the entire preflight request with:

```http
HTTP/1.1 400 Bad Request
Content-Type: text/plain; charset=utf-8

Disallowed CORS headers
```

### Example Failing Request Headers:
```http
Access-Control-Request-Headers: x-token, , authorization
Access-Control-Request-Headers: x-token, authorization, 
Access-Control-Request-Headers: , x-token, authorization
```

Even though both `x-token` and `authorization` were explicitly permitted by the application's CORS configuration, the request was unfairly rejected.

---

## 🛠️ Root Cause Analysis

In `starlette/middleware/cors.py` (inside method `preflight_response`):

```python
# PREVIOUS BUGGY CODE:
elif requested_headers is not None:
    for header in [h.lower() for h in requested_headers.split(",")]:
        if header.strip() not in self.allow_headers:
            failures.append("headers")
            break
```

1. `requested_headers.split(",")` splits on commas.
2. An empty element (e.g. from `", "` or trailing `", "`) produces `" "` or `""`.
3. `header.strip()` produces an empty string (`""`).
4. Since `""` is never in `self.allow_headers`, the condition `if "" not in self.allow_headers` evaluated to `True`.
5. `failures.append("headers")` was executed, causing a `400 Disallowed CORS headers` rejection.

### RFC 9110 Standard (Section 5.6.1.2)
RFC 9110 specifies:
> *"A recipient MUST accept a reasonable number of empty list elements and whitespace."*

HTTP list parsing must gracefully ignore empty list elements rather than failing request validation.

---

## 🧪 Test Case Reproduction

We added a dedicated test case to [`tests/middleware/test_cors.py`](file:///Users/apple/Desktop/jmd/starlette/tests/middleware/test_cors.py):

```python
def test_cors_preflight_allows_empty_elements_in_request_headers(
    test_client_factory: TestClientFactory,
) -> None:
    def homepage(request: Request) -> PlainTextResponse:
        return PlainTextResponse("OK")

    app = Starlette(
        routes=[Route("/", endpoint=homepage)],
        middleware=[
            Middleware(
                CORSMiddleware,
                allow_origins=["https://example.org"],
                allow_headers=["X-Token", "Authorization"],
            )
        ],
    )

    client = test_client_factory(app)

    test_headers = [
        "X-Token, , Authorization",
        "X-Token, Authorization, ",
        ", X-Token, Authorization",
        "X-Token,   ,   Authorization",
    ]
    for requested_headers in test_headers:
        headers = {
            "Origin": "https://example.org",
            "Access-Control-Request-Method": "GET",
            "Access-Control-Request-Headers": requested_headers,
        }
        response = client.options("/", headers=headers)
        assert response.status_code == 200
        assert response.text == "OK"
```

### Test Failure Result (Before Fix):
```bash
E   assert 400 == 200
E   where 400 = <Response [400 Bad Request]>.status_code
tests/middleware/test_cors.py:663: AssertionError
2 failed, 36 deselected in 0.19s
```

---

## Solution Approach
1. **Standards-First Diagnosis:** Verified the HTTP specification (RFC 9110 §5.6.1.2), which explicitly states that recipients MUST parse and ignore empty elements in comma-separated header lists (e.g., `Access-Control-Request-Headers: , content-type, `).
2. **Minimal, Targeted Sanitization:** Rather than relaxing CORS allow-list validation globally (which would weaken security), filter out empty/whitespace-only tokens immediately after splitting `Access-Control-Request-Headers` in `CORSMiddleware.preflight_response`.
3. **TDD & Regression Safety:** Added a dedicated test covering leading, trailing, consecutive, and whitespace-only commas before applying the fix, then verified the full CORS middleware test suite passes without affecting disallowed-header rejection.

## 💡 Solution & Implementation

We updated the list comprehension to strip each element and filter out empty strings (`if h.strip()`) before lowercasing and checking against `self.allow_headers`:

```diff
--- a/starlette/middleware/cors.py
+++ b/starlette/middleware/cors.py
@@ -124,8 +124,8 @@ class CORSMiddleware:
         if self.allow_all_headers and requested_headers is not None:
             headers["Access-Control-Allow-Headers"] = requested_headers
         elif requested_headers is not None:
-            for header in [h.lower() for h in requested_headers.split(",")]:
-                if header.strip() not in self.allow_headers:
+            for header in [h.strip().lower() for h in requested_headers.split(",") if h.strip()]:
+                if header not in self.allow_headers:
                     failures.append("headers")
                     break
```

---

## ✅ Verification & Results

1. **Pytest Run:**
   ```bash
   uv run pytest tests/middleware/test_cors.py
   ```
   **Output:** `38 passed in 0.77s`

2. **Ruff Linter:**
   ```bash
   uv run ruff check starlette/middleware/cors.py tests/middleware/test_cors.py
   ```
   **Output:** `All checks passed!`

3. **Git Commit:**
   `fix(cors): ignore empty elements in Access-Control-Request-Headers per RFC 9110`
