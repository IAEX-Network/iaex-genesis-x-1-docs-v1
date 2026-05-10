# Error Handling

## Public Error Envelope

All public errors use this JSON envelope:

```json
{
  "error": {
    "code": "invalid_api_key",
    "message": "invalid or revoked API key",
    "request_id": "82a1d3f4-17fe-4bf0-9e57-502da6477ff1"
  }
}
```

`request_id` is returned in the `X-Request-Id` response header and echoed in the JSON body when available.

## Common Categories

- Authentication failure: the request did not present a valid API key or valid explicit actor identity.
- Permission failure: the caller is authenticated but does not have access to the requested scope.
- Expired access: a time-bound grant exists but is no longer valid at request time.
- Invalid signature: the actor signature is missing or does not verify for the acting actor.
- Integrity mismatch: the returned event sequence failed integrity verification.
- Not found: the requested public resource does not exist or is not public at that path.
- Not implemented: the path or namespace exists but is not available in this launch version.
- Bad request: the request body, path value, or query parameter is malformed or incomplete.

## Request Correlation

Clients should always log:

- HTTP status
- `error.code`
- `X-Request-Id`

## Idempotency Errors

If a client reuses an `Idempotency-Key` for a different method or path, the server returns an error and does not execute the second request.
