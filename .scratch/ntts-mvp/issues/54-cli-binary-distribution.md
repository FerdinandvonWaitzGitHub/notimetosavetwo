Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

CLI single-binary distribution (F-CLI-8). Primary: `bun --compile` for `linux-x64`, `linux-arm64`, `darwin-x64`, `darwin-arm64`, `windows-x64`; installed via `curl -fsSL https://get.ntts.dev | sh` (downloads platform binary into `/usr/local/bin/ntts`, `chmod +x`, prints version). Secondary: `@ntts/cli` on npm for Node-native CI (`npx @ntts/cli ...`). Both built from one TS codebase under `apps/cli/`. `ntts upgrade-self` checks GitHub Releases for newer binaries and replaces in-place (atomic rename); falls back to print-instructions on Windows.

## Acceptance criteria

- [ ] `bun --compile` produces standalone binaries for all 5 target platforms
- [ ] `curl -fsSL https://get.ntts.dev | sh` installs platform-appropriate binary
- [ ] `@ntts/cli` npm package published; `npx @ntts/cli status` works
- [ ] `ntts upgrade-self` atomic-renames the binary in place on Unix; prints instructions on Windows
- [ ] Single TS codebase under `apps/cli/`; CI publishes both forms
- [ ] All F-CLI-6 command groups wired to backend before final ship

## Blocked by

- [05-migration-apply-and-schema-cache-reload.md](./05-migration-apply-and-schema-cache-reload.md)
