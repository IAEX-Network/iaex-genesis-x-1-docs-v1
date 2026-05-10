# Developer Onboarding Checklist

- Obtain a sandbox and production API key through `POST /developer/register`.
- Record the production actor id and IAEX actor URI returned by registration.
- Enroll an Ed25519 public signing key for each actor that will authorize events.
- Implement actor-signature generation for caller-signed writes.
- Onboard the required organizations through `POST /onboarding/organization`.
- If the integration is entity-native, onboard entities through `POST /entities/onboard`.
- Create required root ledgers and relationship ledgers.
- Create at least one traceledger master and append at least one signed event.
- Confirm event reads return `integrity.verified=true`.
- Resolve at least one actor through `/resolve/{iaex_uri}`.
- Register and verify at least one webhook subscription.
