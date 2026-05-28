# Strike Staking

[Strike Finance](https://www.strikefinance.org/) is a decentralised derivatives platform on Cardano, offering perpetual futures, options, and other structured products. The STRIKE token is the platform's governance and incentive token; holders can lock STRIKE into the protocol's staking contract to earn a share of trading-fee revenue paid out in ADA.

This tx3 covers the **staking product** specifically: locking STRIKE tokens to begin earning rewards, topping up an existing stake, and withdrawing both principal and accumulated rewards. The derivatives trading surface itself is not in scope.

## Overview

When a user first stakes, the protocol mints a credential NFT pair — a *tracker* token (the staking position itself) plus an *owner-identifier* NFT keyed to the staker's payment key hash. Both tokens are locked together in the script UTxO; the owner NFT is **not** returned to the staker's wallet. Ownership is proven on subsequent spends by matching the required transaction signer against the `owner_address_hash` stored in the datum.

Top-ups (`add_stake`) preserve the credential NFTs in the new script output. Withdrawal (`withdraw_stake`) burns both the tracker and the owner NFT, releasing the STRIKE principal and accumulated ADA rewards back to the staker's wallet.

## Transactions

| Transaction | Description |
|---|---|
| `stake` | Lock STRIKE tokens and mint credential NFTs |
| `add_stake` | Add more STRIKE to an existing stake position |
| `withdraw_stake` | Reclaim all STRIKE + accumulated ADA rewards, burning credential NFTs |

## Important considerations

- **Network profile required.** The `env` block references on-chain addresses, policy IDs, and reference-script UTxOs that must be configured per network in the trix profile.
- **Reference scripts.** Both spend and mint validators are consumed via reference scripts (`spend_script_ref`, `mint_script_ref`). The referenced UTxOs must exist on-chain.
- **Owner NFT stays in the script.** Minted with the staker's payment key hash as the asset name and locked alongside the tracker token. `add_stake` preserves it; `withdraw_stake` burns it.
- **Collateral.** All transactions require 5 ADA collateral from the staker.
- **Validity window.** `stake` uses `tip_slot() + 200`.

## Caller preparation

### `stake` / `add_stake`

| Parameter | Source |
|---|---|
| `staked_at_time: Int` | Current Unix time × 1000. tx3's `slot_to_time()` returns seconds, so milliseconds must be computed externally. |

### `add_stake` / `withdraw_stake`

| Parameter | Source |
|---|---|
| `staking_utxo: UtxoRef` | The existing staking position UTxO; queried on-chain by looking up the tracker token at the spend script address. |

## References

- **Smart contracts:** PlutusV3 — [strike-finance/staking-smart-contracts](https://github.com/strike-finance/staking-smart-contracts)
- **Homepage / app:** [strikefinance.org](https://www.strikefinance.org/)
