# Single gateway (`ntts-edge`), JWT validated once

All external traffic enters NTTS through one component — `ntts-edge` — built on Caddy with a custom plugin layer for JWT validation, request-ID generation, rate-limit, and service-role detection. PostgREST, GoTrue, Storage, Realtime, Edge Runtime, and the Studio backend sit on an internal Docker network unreachable from outside. JWTs are validated **once**, at the gateway; upstreams trust the gateway and read identity from internal headers.

## Considered Options

- **Each upstream validates its own JWT.** Rejected: 3–4× the validation work per request, duplicated secret distribution, ambiguous trust boundary, harder to reason about service-role bypass.
- **Three separate components (Caddy + gateway + control-plane).** Rejected: more containers, more inter-hop latency, three places to keep request-ID logic in sync, no real benefit — they all need the same JWT key anyway.
- **Single `ntts-edge` (chosen).** One trust boundary, one place to audit, one place to rate-limit, one place that holds the JWT secret. Upstreams are dumb, internal, and trust the gateway.

## Consequences

- PostgREST is configured with `PGRST_JWT_SECRET=""` and reads role from an internal header (`X-NTTS-Role`) the gateway sets. PostgREST is unreachable from outside the internal network — by design.
- The service-role key exists **only** in the gateway and in the operator's env. It is never distributed to SDKs, frontends, or other containers.
- Defense-in-depth weakens slightly: if `ntts-edge` is compromised, a request with arbitrary role reaches PostgREST. Mitigated by single-node trust, mTLS internal (optional), and the appliance's blast-radius assumption (one Deployment per operator).
- TLS termination lives in `ntts-edge` too; certs persisted in a dedicated `/data/caddy` volume.
- Rate-limit state lives in Postgres (`_ntts.rate_limit_buckets`), not Redis — keeps the appliance promise of "no new container per feature" intact.
