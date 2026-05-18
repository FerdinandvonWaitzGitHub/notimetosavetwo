Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

CI/CD per §4.7a. **CI**: GitHub Actions. **Image Registry**: GHCR primary (`ghcr.io/ntts/*`), Docker Hub secondary mirror. **6 Component Images**: `ntts-edge`, `ntts-studio-backend`, `ntts-typegen`, `ntts-egress-proxy`, `ntts-postgres:slim`, `ntts-postgres:full`. **Tag scheme**: `:v1.3.2`, `:v1.3`, `:latest`, `:edge`. **Multi-arch**: linux/amd64 + linux/arm64 via buildx. **Versioning**: changesets (monorepo internal package versions) + release-please (release-PR automation). **Cadence**: Minor monthly, Patch on-demand, `:edge` on every main-merge. **Compose pinning**: `docker/compose.yml` uses `${NTTS_VERSION:-latest}`. **Image signing**: Cosign/sigstore per image; `ntts upgrade` verifies signatures before pull. Mobile SDK publish lockstep with minor release (F-SDK-5).

## Acceptance criteria

- [ ] All 6 component images build for amd64 + arm64 via buildx in CI
- [ ] Images published to GHCR primary + Docker Hub mirror with the four tag forms
- [ ] Every image signed with cosign on publish
- [ ] `:edge` tag updates on every main-merge
- [ ] changesets + release-please drive version bumps + release PRs
- [ ] Mobile SDK release jobs trigger on NTTS-minor publish (slice 56 hook)
- [ ] CI gate on: lint + tsc + unit + integration + smoke

## Blocked by

- [01-compose-greens-up.md](./01-compose-greens-up.md)
