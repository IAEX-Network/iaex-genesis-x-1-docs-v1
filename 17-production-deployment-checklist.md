# Production Deployment Checklist

- Auth configured: all production calls use valid API keys or the correct explicit actor identity path.
- Signatures verified: caller-signed write requests include valid `X-Actor-Sig` headers where required.
- Resolver reachable: `/resolve/{iaex_uri}` and `/.well-known/iaex-authority` are reachable from verification systems.
- Webhooks tested: a subscription is active, a delivery succeeded, and receiver-side signature verification is enforced.
- Integrity verification working: event consumers check the returned `integrity` block on chain reads.
- Access control tested: direct-party, delegated, expired, and revoked behaviors were exercised in non-production before launch.
- Owner vs delegate behavior verified: owner grants and delegate limitations were confirmed for the scopes your integration uses.
- Public and private boundary verified: no private secrets, private keys, or internal-only assumptions are present in client logs or payloads.
- Sample flows tested end to end: buyer and supplier, entity root, traceledger master, AI response, webhook, and verification paths were all run successfully.
