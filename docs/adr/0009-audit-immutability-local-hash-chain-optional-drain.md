# Audit immutability: local hash-chain default, external drain optional

Audit tables (`_ntts.studio_audit_write`, `_ntts.studio_audit_read`, `_ntts.function_audit`) are append-only at the application role level and tamper-evident via a hash-chain. Operator-with-shell remains the trust root — this is the appliance promise. Operators who need stronger guarantees enable an external drain (`NTTS_AUDIT_DRAIN`) that ships every audit row to S3-object-lock, syslog, or a SIEM in real time.

## Considered Options

- **Triggers + grants only.** Rejected: superuser can drop triggers; no detection capability after the fact.
- **External-drain mandatory.** Rejected: kills "docker compose up and run" promise; not every Operator has a SIEM.
- **HSM / hardware enforcement.** Rejected: incompatible with appliance shipping model.
- **Local hash-chain + optional drain (chosen).** Default gives tamper-evidence without external dependencies; opt-in drain gives tamper-prevention for Operators who genuinely need it.

## Mechanism (default)

- `GRANT INSERT, SELECT` on audit tables to all application roles. **No** `UPDATE`/`DELETE` to any role except `postgres` superuser.
- `BEFORE UPDATE OR DELETE` trigger raises `EXCEPTION 'audit table is append-only'`.
- Each row has `id BIGSERIAL`, `prev_hash`, `payload`, `payload_hash`. `payload_hash = sha256(prev_hash || canonical_json(payload))`.
- Audit-row INSERTs serialise via `pg_advisory_lock(audit_table_oid)` to keep the chain consistent under concurrent writers.
- `_ntts.verify_audit_chain(table_name)` SQL function traverses the chain; CLI exposes `ntts audit verify`.

## Consequences

- The §7 NFR row "Audit log immutable" becomes honest: *"INSERT-only at app-role level, hash-chained for tamper detection. Postgres superuser is the trust root; external drain available for stronger guarantees."*
- Operator-with-shell can clear audit by dropping triggers + truncating. Documented; matches the appliance trust model.
- Hash-chain validation is a periodic operation (CLI + Studio "Audit Health" panel), not blocking on write.
- PII in audit rows is *redacted at row creation* — audit captures the *fact* of access, not the *content* (e.g. "Operator X opened auth.users list at T", not the list itself). F-STU-7 / F-STU-8 enforce this.
- GDPR Hard-Delete (F-GDPR-3) leaves audit rows intact — they record Operator actions, not App User data. Aligns with DSGVO accountability requirements.
