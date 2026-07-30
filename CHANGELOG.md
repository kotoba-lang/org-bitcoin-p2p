# Changelog

## 0.4.0 — 2026-07-30

- Add Bitcoin Core v31.1 testnet4 and default signet proof-of-work parameters
  and hard-coded genesis trust anchors.
- Implement BIP94 testnet4 retargeting from the first block of each difficulty
  period while preserving the minimum-difficulty exception between periods.

## 0.3.0 — 2026-07-30

- Add Bitcoin Core-compatible regtest genesis, magic, port, proof limit, and
  no-retarget header consensus for deterministic fork/reorg testing.

## 0.2.0 — 2026-07-30

- Enforce mainnet and testnet3 expected difficulty, including bounded
  retargeting and testnet minimum-difficulty recovery.
- Enforce median-time-past and the two-hour future timestamp bound.
- Add exact 256-bit header work, cumulative chainwork, and more-work fork
  comparison.
- Persist height and chainwork atomically with validated header batches.
- Reject wrong-network magic, oversized messages, more than 2,000 headers,
  non-zero header transaction counts, truncation, and trailing data.
- Refuse legacy stored tips without chainwork instead of silently assigning an
  incorrect work value.

This remains a read-only headers client. It does not validate transactions,
Merkle roots, Script, or the UTXO set and must not be described as a full node.
