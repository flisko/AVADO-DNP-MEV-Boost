# Glamsterdam / ePBS migration plan

How the AVADO MEV setup changes when Glamsterdam activates, and what has to be
built before then. This package (`mevboost.avado.dnp.dappnode.eth`) is retired
by that fork.

**Status: not urgent, not yet actionable in full.** Glamsterdam has no mainnet
activation in any client, and the specs this plan depends on are still open PRs.
Treat the file layouts below as provisional until they merge.

---

## 1. Where things actually stand

Verified against client source on 2026-08-05, not press coverage:

| Client | Version checked | Glamsterdam state |
|---|---|---|
| Prysm | v7.1.8 | `mainnetGloasForkEpoch = math.MaxUint64` — unscheduled |
| Teku | 26.7.1 | `GLOAS_FORK_EPOCH: 18446744073709551615` — unscheduled |
| Geth | v1.17.5 | `AmsterdamTime` exists in `ChainConfig`, unset on mainnet |
| eth-clients/mainnet | current | no `GLOAS_FORK_EPOCH` key at all; latest is `FULU_FORK_EPOCH` |
| mev-boost | v1.12 | no ePBS support — zero references to gloas/amsterdam/epbs/7732 |

Glamsterdam = **Gloas** (consensus layer) + **Amsterdam** (execution layer).
Prysm's "Gloas" work throughout the v7.1.x series is the ePBS implementation.

The readiness signal to watch for is a client release that ships an actual fork
epoch or timestamp. Until then nothing can be verifiably "Glamsterdam-ready".

### Re-checking readiness

```sh
# CL: is Gloas scheduled on mainnet yet?
curl -s https://raw.githubusercontent.com/eth-clients/mainnet/main/metadata/config.yaml \
  | grep -E 'GLOAS_FORK_EPOCH|FULU_FORK_EPOCH'

# Prysm: MaxUint64 (18446744073709551615) means unscheduled
curl -s https://raw.githubusercontent.com/OffchainLabs/prysm/<tag>/config/params/mainnet_config.go \
  | grep -n 'mainnetGloasForkEpoch'

# EL: AmsterdamTime present in MainnetChainConfig?
curl -s https://raw.githubusercontent.com/ethereum/go-ethereum/<tag>/params/config.go \
  | awk '/MainnetChainConfig = &ChainConfig\{/,/^\}/' | grep -i amsterdam
```

---

## 2. What changes

Today (PBS via sidecar):

```
VC  --(--enable-builder)-->  BN  --(--http-mev-relay)-->  mev-boost  -->  relays
```

The relay list, `-min-bid` and friends live in the **sidecar**. The VC only says
"builder enabled" and supplies a fee recipient.

After Glamsterdam (ePBS, in-protocol):

```
VC  --(per-key builder prefs)-->  BN  --(direct)-->  builders
```

The **validator client owns the builder list**, per key: which builders to
request bids from, the auth data agreed with each, `max_execution_payment`,
`min_bid`, `builder_boost_factor`. The VC must own it because it pre-signs the
request authentications binding each request to that builder's auth data.

There is no sidecar in this picture. mev-boost / commit-boost-pbs is removed.

### The three specs defining the flow

| Link | Defines |
|---|---|
| [keymanager-APIs#87](https://github.com/ethereum/keymanager-APIs/pull/87) | User ↔ VC configuration |
| [beacon-APIs#630](https://github.com/ethereum/beacon-APIs/pull/630) | VC ↔ BN communication |
| [builder-specs#165](https://github.com/ethereum/builder-specs/pull/165) | BN ↔ Builder communication |
| [prysm#17124](https://github.com/OffchainLabs/prysm/pull/17124) | Prysm's static-file form of the same config |

All four are **open** as of writing.

---

## 3. keymanager-APIs#87 — `/eth/v1/validator/config`

Adds one endpoint (`GET`/`POST`) managing a validator's whole block-production
preference set as **one atomic per-key document**, and **deprecates all 9
operations** across the existing `feerecipient`, `gas_limit` and `graffiti`
endpoint families.

Per key: `{fee_recipient, target_gas_limit, graffiti, builder}`, where `builder`
holds `enabled` and a `builders[]` list. Each entry carries `url`, `auth_data`,
`max_execution_payment`, `min_bid`, `builder_boost_factor` (all required) plus an
optional `pubkey` pinning the expected bid signer.

Semantics that matter for the wizard:

- **POST is full-replace per key, never a merge.** Submit a complete document, or
  `{}` to clear the key back to defaults. There is no DELETE.
- **Nothing is inherited or merged per field.** A key either has a complete
  document used exactly as stored, or none, in which case it follows
  `default_config` *whole*.
- **Entries are identified by the `(url, auth_data)` pair.** Multiple entries may
  share a `url` with different `auth_data`; one request is sent per entry.
  Duplicate pairs reject that key's entire document.
- **Atomicity is per key, not per batch.** Valid keys apply even if others in the
  same request fail; outcomes are reported per key.
- `default_config` is **read-only** through this API in this version.
- Monetary values are Gwei as decimal strings.

Because the API is all-or-nothing per key, the wizard has to resolve a *complete*
document client-side before submitting — it cannot patch one field the way the
current fee-recipient endpoint allows. That is the main UI change.

---

## 4. Impact per AVADO package

### `mevboost.avado.dnp.dappnode.eth` — retired

Removed after the fork. Do not delete it before: it must keep serving until the
fork lands, and users need a working sidecar for any pre-fork rollback.

### `eth2validator.avado.dnp.dappnode.eth` — most of the work

`build/files/start.sh` already generates `/root/.eth2validators/proposer_settings.json`,
today in the **v1** shape:

```jsonc
{ "default_config": { "fee_recipient": "0x…", "builder": { "enabled": true } } }
```

Prysm v7.1.6 introduced **v2** proposer settings. Relevant already-shipped changes:

- The `builder` option is not recognised post-Gloas; `gas_limit` moves to the top
  level of proposer preferences.
- v1 files still work: a deprecation warning is logged on Gloas-scheduled
  networks and settings are upgraded automatically at the fork, promoting builder
  gas limits to the top level. Files already on v2 are never rewritten.
- v7.1.7 removed the top-level `max_execution_payment` in favour of the
  per-builder `BuilderConfig.max_execution_payment`.

So the file AVADO writes keeps working through the fork, but stops expressing
anything useful — `builder.enabled` alone means nothing without a builder list.

Target shape (per prysm#17124):

```json
{
  "version": 2,
  "default_config": {
    "fee_recipient": "0x…",
    "gas_limit": "45000000",
    "builder": {
      "enabled": true,
      "builders": [
        { "url": "https://builder-a.example.com", "min_bid": "10000000", "builder_boost_factor": "100" }
      ]
    }
  }
}
```

Work items:

1. Teach `start.sh` to emit `version: 2` with a `builders[]` list built from a new
   settings field, replacing today's `builder.enabled` toggle.
2. Extend the wizard: a builder-list editor (URL, optional auth data, min bid,
   max execution payment, boost factor) replacing the current MEV on/off switch.
3. Drop `--enable-builder`; it is the v1 toggle.
4. If the wizard moves to the keymanager API rather than writing the file
   directly, implement `/eth/v1/validator/config` and retire its use of the
   `feerecipient` / `gas_limit` / `graffiti` endpoints together — PR 87
   deprecates all three families at once.

### `prysm-beacon-chain-mainnet.avado.dnp.dappnode.eth`

`build/startPrysmBeaconchain.sh` passes:

```sh
${MEV_BOOST_ENABLED:+--http-mev-relay=http://mevboost.my.ava.do:18550}
```

Remove after the fork. The BN dials builders directly using the `url` values
forwarded from the VC, and routes by `url` only — auth data is forwarded
byte-for-byte.

### Geth / Nethermind

No MEV-specific work. They need an Amsterdam-capable release before the fork like
any other EL upgrade.

---

## 5. Sequencing

**Now → fork scheduled.** No package changes. Watch for a client release carrying
a real Gloas epoch. Keep clients current. Note that a proxy path exists —
prysm#17124 states you can point a builder `url` at a sidecar and use `auth_data`
to identify the downstream builder — which is worth evaluating as a transition
option if it survives review.

**Fork scheduled → fork.** Ship the VC-side builder config (items 1–3 above) so
users can configure builders *before* activation. Ship a validator release whose
Prysm version has a non-MaxUint64 Gloas epoch. Keep mev-boost running and
unchanged. Decide the default builder set — the relay list in
`dappnode_package.json` does **not** carry over; relay URLs are not builder URLs.

**At the fork.** Clients switch from sidecar PBS to ePBS automatically. Nothing to
do if the config landed beforehand.

**After the fork.** Remove `--http-mev-relay` from the BN, remove
`--enable-builder` from the VC, deprecate and then delete this package. Leave a
release that no-ops with a clear log line rather than breaking installs that
still have it.

---

## 6. Open questions

- **Default builder set.** mev-boost shipped a curated relay list. Is there an
  equivalent for builders, and should AVADO ship defaults at all? Relay URLs are
  not reusable here.
- **Wizard vs keymanager API.** Writing `proposer_settings.json` directly is less
  work; the keymanager API is the portable path and what upstream tooling will
  target. PR 87 deprecating 9 existing operations is an argument for moving.
- **Existing installs.** Users have a persisted `mev_boost` boolean in
  `settings.json`. Needs a defined meaning post-fork — probably "builder enabled
  with an empty builder list", which is inert until they configure one.
- **Rollback.** Once `--http-mev-relay` is gone from a released BN, reverting
  means reinstalling the older package. Worth a documented path.

---

## 7. References

- [keymanager-APIs#87](https://github.com/ethereum/keymanager-APIs/pull/87) — `/eth/v1/validator/config`
- [beacon-APIs#630](https://github.com/ethereum/beacon-APIs/pull/630) — VC ↔ BN
- [builder-specs#165](https://github.com/ethereum/builder-specs/pull/165) — BN ↔ Builder
- [prysm#17124](https://github.com/OffchainLabs/prysm/pull/17124) — per-builder entries in proposer settings
- [prysm#16762](https://github.com/prysmaticlabs/prysm/pull/16762) — proposer settings v2, shipped in v7.1.6
- EIP-7732 (ePBS), EIP-7928 (Block-Level Access Lists) — Glamsterdam headline EIPs
