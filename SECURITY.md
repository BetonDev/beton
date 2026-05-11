# Security Policy

## Scope

This policy covers the deployed `beton` Solana program and the security metadata published for Beton at https://github.com/BetonDev/beton.

## Reporting A Vulnerability

Report vulnerabilities privately to betondev@proton.me.

Include the affected program, network, program ID, reproduction steps, expected impact, and any proof of concept needed to validate the issue.

Do not publicly disclose the issue before a fix is available.
Do not access, move, or place user funds at risk.
Do not degrade service availability or perform denial-of-service testing against production infrastructure.

Valid reports are reviewed and handled as coordinated disclosures. Any acknowledgements or bug bounty payouts are at the project's discretion.

## Disclosure Process

After validating a report, the project will work toward a fix and coordinate public disclosure once remediation is available.

## Operational Hardening (Mainnet)

The following operational invariants are part of the production security model
and MUST hold before each mainnet deploy or upgrade of `beton`.

### Upgrade authority

The deployed `beton` program retains a single offline upgrade authority. That
key is not a day-to-day operator credential.

Rules:

1. Use the upgrade authority only for audited program deploys, authority
   rotation, or final authority revocation.
2. Whenever the upgrade authority is brought online and used, transfer program
   upgrade authority immediately to a fresh offline key.
3. Never use the upgrade authority as the Beton privileged operator.

Verification:

```sh
solana program show BpwBgBZ8WFDk7BswjJNoepmisZS1KfuoKbybCLaF5Hj6 --url mainnet-beta
```

### Beton privileged operator

`beton` hardcodes a single separate privileged operator pubkey inside the
program binary. That operator controls Beton-only admin flows such as
`init_protocol`, jackpot bootstrap, reward bootstrap, reward treasury
management, and reward campaign creation, launch, and expiry.

Rules:

1. The Beton operator key MUST be separate from the current Beton upgrade
   authority.
2. Release operators MUST verify the expected operator public key before any
   Beton admin action.
3. If the Beton operator key is compromised, rotate it only through an audited
   Beton program upgrade.
4. Public documentation intentionally does not describe private key-handling or
   rotation ceremonies beyond these invariants.

### No pause mechanism

The current production posture intentionally does not include a pause switch or
reward-campaign emergency halt.

Rules:

1. Reward campaigns rely on immutable launch budgets, permissionless claims,
   and permissionless expiry after the fixed deadline.
2. Incident response falls back to the existing operator and upgrade-authority
   model and, if needed, an audited program upgrade.

### Browser RPC ingress

The public browser RPC path MUST terminate behind a reverse proxy that
overwrites the configured `TRUSTED_CLIENT_IP_HEADER` before forwarding to the
Next.js app. Client-supplied copies of that header must be stripped.

Production browser traffic must use `/api/solana-rpc-public` for read and
simulation methods only. The protected `/api/solana-rpc` route is reserved for
server-side checks and operator tooling authenticated by
`SOLANA_RPC_PROXY_TOKEN`.

### IDL pinning

The Beton IDL published to chain (`anchor idl init` / `anchor idl upgrade`)
MUST match the git-tagged release used for the on-chain binary,
byte-for-byte. CI must fail if `anchor build` produces a diff against the
committed reviewed IDL artefact. Off-chain clients should pin to the reviewed
release IDL, not the chain-fetched one, until each upgrade is audited.

### Pre-deploy checklist

Before each mainnet `solana program deploy` or `anchor program upgrade` of
`beton`:

1. `cargo test -p beton --lib` and `pnpm run test:beton:mollusk` pass on the
   exact commit being shipped.
2. The shipped `anchor/target/deploy/beton.so` artifact was produced from a
   clean checkout of the same commit.
3. The shipped commit hash is recorded alongside the on-chain program-data
   slot in the release ledger.
4. Release operators have re-verified the separate Beton operator key before
   running any Beton admin instruction on mainnet.
