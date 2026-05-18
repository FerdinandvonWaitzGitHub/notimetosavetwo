# ADR-0028: Master-Key-Verfügbarkeit während Upgrade-Apply

**Status:** Accepted
**Date:** 2026-05-18
**Resolves:** Design-Lücke in ADR-0011 (Master-Key-Verfügbarkeit für re-encryption migrations)

## Context

`NTTS_MASTER_KEY` ist der pgcrypto-Master per CONTEXT.md und F-FN-10: **lebt nur in der Edge-Runtime-Container-Env, niemals in `_ntts`**. Verschlüsselt:

- `_ntts.function_secrets` (Function-Secrets, F-FN-10)
- `_ntts.platform_secrets` (HMAC-Keys F2F + Gateway-Realtime per §4.10, Postgres-Superuser-Password per F-DB-1, OAuth client_secrets, SMS credentials, Wrapper credentials, TLS BYO-cert keys, AI provider key per F-VLT-4)

Eine NTTS-internal-Migration die diese Daten re-encrypten muss (z.B. pgcrypto-Algorithmus-Bump, libsodium-Migration, Key-Wrapping-Format-Change), braucht **lese-Zugriff auf den aktuellen Master-Key und schreibe-Zugriff für den neuen Verschlüsselungs-State**.

Während eines `ntts upgrade` ist die Edge-Runtime aber gestoppt (per ADR-0011 step 5). Der Master-Key, der nur dort gelebt hat, ist nicht mehr im Prozess-Memory verfügbar. Drei Optionen wurden in der Grilling-Session 2026-05-18 betrachtet:

## Considered Options

1. **Operator-prompted Master-Key-Eingabe.** Operator gibt den Master-Key beim Upgrade explizit via CLI.
   - **Akzeptiert.** Sauberster Trade-off: scoped auf upgrade-Operation, bricht F-FN-10-Contract ("nur Edge-Runtime kennt Master-Key") nur kurz und kontrolliert.

2. **studio-backend hält Master-Key dauerhaft.** Würde bedeuten, dass studio-backend `NTTS_MASTER_KEY` als Env hat.
   - **Verworfen:** widerspricht F-FN-10 ("lives only in the Edge Runtime container env"). Erweiterung des Trust-Surfaces für ein selten-genutztes Feature.

3. **Edge-Runtime materialisiert Master-Key kurz vor Shutdown.** Edge-Runtime schreibt Master-Key in tmpfs, kurz vor `docker compose down`. Upgrade-Apply-Container mountet tmpfs.
   - **Verworfen:** komplizierter Container-Lifecycle, Race-Conditions zwischen Edge-Runtime-Shutdown und Upgrade-Apply-Start, tmpfs würde Master-Key auf disk-shaped-Storage halten falls tmpfs swap fällt (Linux tmpfs swap-out möglich bei Memory-Druck).

## Decision

**Operator-prompted Master-Key-Pass-Through, scoped auf upgrade-apply, tmpfs-materialisiert für Migration-Dauer.**

### Migration-Manifest

NTTS-internal Migrationen tragen Metadata im File-Header:

```sql
-- ntts:requires_master_key=true
-- ntts:reason=pgcrypto-algorithm-upgrade-aes256-to-libsodium-secretbox
-- ntts:affects=_ntts.platform_secrets,_ntts.function_secrets
CREATE OR REPLACE FUNCTION _ntts_internal.reencrypt_secrets() RETURNS void AS $$
...
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

`_internal upgrade-prep` parst alle pending Migrationen, akkumuliert `requires_master_key`-Flags. Wenn mindestens eine pending Migration den Master-Key braucht, setzt prep `requires_master_key=true` im Pre-Check-Output.

### CLI-Flow

`ntts upgrade --check` warnt: *"This upgrade requires NTTS_MASTER_KEY for re-encryption of N secrets. You will be prompted during apply."*

`ntts upgrade [--to vX.Y]` läuft den Standard-Flow aus ADR-0011 bis Step 5 (upgrade-apply). Wenn `requires_master_key=true`:

- Bei interaktivem Terminal: `getpass`-Style-Prompt *"Enter NTTS_MASTER_KEY:"*, Echo unterdrückt, Eingabe nicht in History.
- Bei non-TTY (CI / Scripted): muss via `--master-key-file <path>` (Datei mit dem Key, mode-Check 0400, owner-Check current user) oder via env `NTTS_UPGRADE_MASTER_KEY` (CLI liest, löscht sofort aus eigenem env nach Read).
- Validation: CLI hashed den eingegebenen Key mit dem in `_ntts.platform_secrets[master_key_fingerprint]` persistierten Fingerprint (SHA-256 des Keys, *nicht* der Key selbst). Mismatch → Upgrade abort.

### Materialisation

- studio-backend (welcher als Orchestrator den Upgrade fährt — siehe ADR-0011) erhält den Master-Key via stdin-Pipe von der CLI über einen authenticated WS-Channel zur Upgrade-Operation.
- studio-backend schreibt den Master-Key in `/var/run/ntts/upgrade/master.key`, mode `0400`, owner `ntts:ntts`, auf tmpfs-Volume `ntts-upgrade-runtime` (`tmpfs,noswap`).
- Upgrade-Apply-Container (das ist eine kurzlebige Variante des neuen NTTS-Images, gestartet mit `--mode upgrade-apply`) mountet `ntts-upgrade-runtime` read-only nach `/var/run/ntts/upgrade/`.
- Upgrade-Apply-Code liest Master-Key aus dieser Datei, leitet ihn in die Migration-DDL via `SET LOCAL ntts_upgrade.master_key = '<key>'` ein. Die re-encryption-Migration ruft `current_setting('ntts_upgrade.master_key')`.
- `SET LOCAL` ist transaction-scoped: nach COMMIT oder ROLLBACK ist der Key aus Session-State weg.

### Post-Apply-Cleanup

- Nach Upgrade-Apply (egal ob COMMIT oder ROLLBACK) ruft studio-backend `unlink('/var/run/ntts/upgrade/master.key')` + `shred -u` (mehrfaches Overwrite). Auf tmpfs effektiv sofortige Memory-Klärung.
- studio-backend prüft mit `fs.access` dass die Datei weg ist. Wenn nicht: ALERT WARN per F-LOG-4, refuse-to-finish-upgrade.
- Master-Key-im-studio-backend-Memory wird mit `Buffer.alloc(0)` overwritten + GC-getriggert. Best-Effort — V8 garantiert kein Memory-Wipe.

### Audit

Ein Master-Key-using-upgrade schreibt:

- `_ntts.studio_audit_write` row: `(action='upgrade_with_master_key_unlock', target_kind='ntts_version', target_id='<new_version>', payload={'migrations_required_key': […], 'fingerprint_verified': true})`.
- `_ntts.function_audit` für jede betroffene Function-Secret-Re-Encryption: `(action='secret_reencrypted', operator_id=<upgrade-runner>, override_reason_category='infrastructure-issue', override_reason_text='ntts upgrade ${version_from}→${version_to} re-encrypted secret')`.

### Failure-Modes

- **Wrong Master-Key entered:** Fingerprint-Check schlägt fehl bevor Upgrade-Apply startet. Upgrade abort, kein State-Change. Snapshot-Rollback nicht nötig.
- **Master-Key korrekt, aber re-encryption-Migration scheitert mid-transaction:** Transaction rollback automatisch (alle pending Migrationen in einer TX per ADR-0011). Snapshot intakt. Standard-rollback-Pfad.
- **Master-Key korrekt, Migration committed, aber Verify-Stage schlägt fehl:** `ntts upgrade --rollback`. Wiederhergestellter Snapshot ist im *alten* Verschlüsselungs-Format. Boot mit dem alten Image, der den alten Algorithmus kennt. Operator-Erfahrung: Upgrade scheinbar zurückgerollt, alle Function-Secrets funktionieren wie vorher.
- **studio-backend stürzt ab nach Master-Key-Materialisierung, vor Upgrade-Apply-Container-Start:** tmpfs-File lebt noch. **Mitigation:** systemd-Cleanup-Hook auf studio-backend container restart räumt `/var/run/ntts/upgrade/` rigoros auf. Backup: tmpfs Lebensdauer ist Pod-Lifetime; bei Compose-Restart wird tmpfs neu erzeugt.

## Consequences

### Implementations-Aufwand

- Migration-Manifest-Parser im upgrade-prep: ~100 LOC Node/TS in studio-backend.
- CLI-Prompt + Validation + WS-Pipe zu studio-backend: ~200 LOC TS in `apps/cli/`.
- tmpfs-Mount + Materialisierung + Cleanup in studio-backend: ~150 LOC TS.
- Fingerprint-Persistierung in `_ntts.platform_secrets[master_key_fingerprint]`: NTTS-internal-Migration 1 Schritt, im Init-Migration (F-UPG-1 `0001_init.sql`).
- Audit-Row-Schreibung: trivial, ~30 LOC.

**Total: ~500 LOC Solo, ~3-5 Tage.**

### Trust-Boundary-Impact

- Master-Key existiert für die Dauer eines Upgrade-Applies im studio-backend Memory + auf tmpfs. Davor und danach nicht.
- F-FN-10-Contract ("lives only in Edge-Runtime container env") wird **kontrolliert temporär gebrochen**. Audit-Row macht den Bruch nachvollziehbar.
- Operator-handled (CLI-Prompt) macht den Bruch sichtbar für den Operator selbst — kein "silent shared secret" zwischen Containern.

### Operator-UX

99% der NTTS-Upgrades brauchen keinen Master-Key (sie ändern Schema, fügen Spalten hinzu, ändern Indizes — nicht Verschlüsselungs-Format). Für diese 99% ist die Operator-UX unverändert.

Für die seltenen Re-Encryption-Upgrades: ein zusätzlicher Prompt-Step. Dokumentation in `RELEASE_NOTES.md` muss markieren "Requires Master-Key for re-encryption."

### Was die Architektur nicht löst

- **Master-Key-Verlust während Re-Encryption-Upgrade:** wenn Operator den Key nicht mehr hat (per Frage 15 Memory: Backup-out-of-band, "tough luck"), kommt er nicht über den Pre-Check. Workaround: vorher manuell `ntts admin rotate-master-key` mit dem alten Key (falls noch im Edge-Runtime-Container vorhanden via `docker inspect`), neuer Key in Operator-Backup, dann Upgrade. Wenn Edge-Runtime-Container schon weg ist → siehe Frage-15-Memory: out-of-band-Recovery fehlt, "tough luck" gilt.
- **Side-Channel-Leakage des Master-Keys während tmpfs-Window:** ein Angreifer mit local-root-Access auf den Host kann tmpfs lesen während des Upgrade-Apply-Windows (~1-2 Sekunden bei kleinen Re-Encryption-Loads, länger bei großen). Mitigation: Threat-Model nimmt an, dass Host-Root nicht kompromittiert ist während eines Upgrades. Wenn doch: weit größere Probleme als der Master-Key.
- **Concurrent-Upgrades:** zwei `ntts upgrade`-Aufrufe parallel ist nicht supported (per ADR-0011 inferiert). Lock-File `/var/run/ntts/upgrade.lock` im studio-backend verhindert das.

## Related

- F-FN-10 (Master-Key in Edge-Runtime-only) — diese ADR definiert den einzigen sanctioned exception case.
- ADR-0007 (Function-Secret-Boot-Injection) — re-encryption-Migrations betreffen die in `function_secrets` lebenden Keys.
- ADR-0008 (Three Classes of Secrets) — diese ADR betrifft `function_secrets` (Class 1) und `platform_secrets` (Class 3); Class 2 (Vault) hat eigenen pgsodium-Server-Key, separat behandelt.
- ADR-0011 (Upgrades) — referenziert diese ADR für Master-Key-Workflows.
- Frage 15 Memory-Entscheidung (Out-of-band-Backup, kein NTTS-Tooling) — diese ADR adressiert nicht Master-Key-Verlust-Recovery; out-of-band bleibt.
