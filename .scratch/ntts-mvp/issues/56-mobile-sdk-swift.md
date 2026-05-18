Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

Mobile SDK: Swift v1.0 via SPM (F-SDK-4, F-SDK-5). Thin re-export of `supabase-swift` with NTTS-generated types surfaced as Swift Codable structs. Versioned lockstep with NTTS-minor via GitHub release tags (`ntts-swift`). CI: every NTTS-minor release rebuilds + tags Swift SDK with signed-tag publish. Kotlin/Flutter/Python deferred post-v1.0.

## Acceptance criteria

- [ ] `ntts-swift` GitHub repo published; SPM target consumable
- [ ] Quickstart Swift app (in `mobile-sdks/swift/quickstart/`) builds + runs + signs a user in (§6 #25)
- [ ] Codable structs generated from NTTS schema match table shape
- [ ] Lockstep versioning: every NTTS-minor release publishes a matching Swift SDK tag with signed publish
- [ ] Code under `mobile-sdks/swift/` in the monorepo with its own toolchain

## Blocked by

- [55-js-sdk-client.md](./55-js-sdk-client.md)
