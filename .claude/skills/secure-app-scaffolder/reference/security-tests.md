# Security Test Matrix

Every generated project ships these tests in `backend/tests/security/`. They run in `make test` and fail the build.

## Authorization (generated over all routes)
Enumerate the application's routes and parametrize, so a new route is tested automatically:

```python
@pytest.mark.parametrize("route", all_api_routes(exclude=PUBLIC_ROUTES))
def test_route_requires_auth(client, route):
    assert client.request(route.method, route.sample_path).status_code == 401

@pytest.mark.parametrize("route", object_routes())
def test_other_user_object_is_hidden(client_as_bob, alice_object, route):
    assert client_as_bob.request(route.method, route.path_for(alice_object)).status_code == 404

@pytest.mark.parametrize("route", admin_routes())
def test_admin_route_forbidden_to_user(client_as_user, route):
    assert client_as_user.request(route.method, route.sample_path).status_code == 403
```

## Required cases
| Area | Case | Expected |
|---|---|---|
| Input | Unknown field (`"role": "admin"`) | 422, field not applied |
| Input | Body over limit | 413 |
| Input | Page size over max | clamped or 422 |
| Idempotency | Same key, same body twice | one write, same response |
| Idempotency | Same key, different body | 422 |
| CSRF | State-changing request without header/token | 403 |
| CSRF | Foreign `Origin` | 403 |
| Auth | Unknown user vs wrong password | same status, message, similar timing |
| Auth | Repeated failures | 429 / progressive delay |
| Auth | Password change | other sessions revoked |
| Auth | Reset token reused or expired | rejected |
| Auth | TOTP code reused | rejected |
| Auth | Admin without MFA | cannot access admin routes |
| Session | Cookie flags | `__Host-`, `HttpOnly`, `Secure`, `SameSite=Lax` |
| Session | Admin idle > 1 h | session expired |
| Redirect | `next=//evil.example` | rejected |
| SSRF | Webhook to `http://169.254.169.254/` or `127.0.0.1` | rejected |
| Export | Cell `=CMD()` | exported as `'=CMD()` |
| Upload | Wrong magic bytes, oversized, path in name | rejected |
| Honeytoken | Access to trap endpoint | IP blocked, critical audit entry |
| Errors | Forced 500 | no stack trace, `request_id` present |
| Headers | Any page | CSP, HSTS, nosniff, frame-ancestors present |
| Logs | Login request | password absent from logs |
| Audit | Audit row modified in DB | hash chain verification fails |
