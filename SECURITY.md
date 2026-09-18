# Security Policy

## Reporting a vulnerability

Email **security@beexar.com**. Please do not open a public issue.

Include the affected package and version, what you observed, and the smallest
reproduction you have. We acknowledge within two business days.

## Scope

These SDKs sign and verify HMAC-SHA256 signatures over raw HTTP bodies and
carry money as decimal strings. Reports we are especially interested in:

- any way to make signature verification accept a body it should reject
- any path where a request body is parsed and re-serialised before verification
- money parsing that accepts a value it should refuse, or loses precision
- a way to construct an `api_code` 100/105/106 error without a balance

## Out of scope

The operator's own ledger, database, and idempotency storage. The SDK
deliberately does not own those — see "The boundary" in each README.
