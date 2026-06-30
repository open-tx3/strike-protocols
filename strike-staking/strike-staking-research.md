# Strike Finance Staking — On-Chain Research

## Protocol identifiers

| Field | Value |
|-------|-------|
| STRIKE policy ID | `f13ac4d66b3ee19a6aa0f2a22298737bd907cc95121662fc971b5275` |
| STRIKE asset name (hex) | `535452494b45` ("STRIKE") |
| STRIKE decimals | 6 |
| STRIKE minting tx | `d11ca3dd03e899fbd1d76da68dac619617dd56372f65ef55e68ced1334f38e08` (block 10755594) |

---

## ⚠️ Active contract (correct)

> **The active contract with the highest volume is NOT the v2 or v3 from the previous sections.**
> The tx3 based on the public GitHub (bkp_main.tx3) points to the CORRECT contract.
> The v2/v3 research below describes legacy contracts with lower activity.

| Field | Value |
|-------|-------|
| Script hash | `497a8b0085517f1c9065cf3006af4c266454b39c6fa32a9d116c75ee` |
| Type | PlutusV3 (3781 bytes) |
| Staking address | `addr1z9yh4zcqs4gh78ysvh8nqp40fsnxg49nn3h6x25az9k8tms6409492020k6xml8uvwn34wrexagjh5fsk5xk96jyxk2qf3a7kj` |
| Reference script UTxO | `486c6c010d1518b1e032d2a288483fba55cee4f054b6e97f4e7eadeccb173768#0` (spend AND mint = same UTxO) |
| mint_policy_id | `497a8b0085517f1c9065cf3006af4c266454b39c6fa32a9d116c75ee` (= script hash) |
| tracker_asset_name | `535452494b45` (= same as STRIKE asset name) |
| Activity | 658+ txs since block 13050000, ~41k indexed redeemers |

### Redeemers confirmed on-chain (active contract)

| Operation | Type | On-chain structure | Occurrences |
|-----------|------|---------------------|-------------|
| `stake` | mint | `Constr(0, [Int])` — amount | 2456 |
| `withdraw_stake` | mint | `Constr(1, [Bytes])` — owner_pkh | 1303 |
| `add_stake` / `consume_rewards` | spend | `Constr(0, [])` | 15764 |
| `withdraw_stake` | spend | `Constr(1, [])` | 1307 |
| `distribute_rewards` | spend | `Constr(2, [])` | 18903 |

This matches **exactly** the code in the public GitHub:
- `MintRedeemer::Mint { amount }` = mint Constr(0)
- `MintRedeemer::Burn { owner_address_hash }` = mint Constr(1)
- `StakingRedeemer::AddStakeOrConsumeStakingRewards` = spend Constr(0)
- `StakingRedeemer::WithdrawStake` = spend Constr(1)
- `StakingRedeemer::DistributeStakingRewards` = spend Constr(2)

### Reference tx hashes (most recent per operation)

| Operation | Tx hash | Block |
|-----------|---------|-------|
| `stake` | `f70239fa91bcb0df496c011465e440e8cd97955231df956c7de3820e1c861a80` | 13140934 |
| `add_stake` | `939737ecd4f1ed2cab613a63606802287e79b0525767ddaff6057bbafd28bfab` | 13140518 |
| `withdraw_stake` | `60f83cf13c421d33a039cd902b57dd67fec4f4afbcdbade5a597b3f816d58b06` | 13140491 |
| `consume_rewards` | `4a458236a0b4074703970fffa32a337314094963df6255e07dd967a690466260` | 13139985 |

---

## Legacy contracts (lower activity)

The protocol has had **3 previous versions** of the staking contract:

| Version | Script hash | Address | Active period |
|---------|-------------|-----------|----------------|
| v1 | `2025463437ee5d64e89814a66ce7f98cb184a66ae85a2fbbfd750106` | `addr1zysz2335xlh96e8gnq22vm88lxxtrp9xdt595taml46szpnqcef7gz6hguyxwz0wuwuq64ryws4ws8pqennhd28rgh8s0nh7pw` | Aug 2024 (a few days) |
| v2 | `932298043638b552288bd1fa2e5d49408eb83e964fecf84c6f2e15a1` | `addr1zxfj9xqyxcut253g30gl5tjaf9qgawp7je87e7zvduhptg275jq4yvpskgayj55xegdp30g5rfynax66r8vgn9fldndsgv4y9t` | Jan 2025 – present |
| v3 | `1af84a9e697e1e7b042a0a06f061e88182feb9e9ada950b36a916bd5` | `addr1zyd0sj57d9lpu7cy9g9qdurpazqc9l4eaxk6j59nd2gkh40vvwe5f7xtt25s5fyftlm468rnjznztvgn9p0gvvr72p5qcl3cq7` | Aug 2024 – present |

---

## Reference Scripts (KEY FINDING)

| Contract | Reference script UTxO | Creation block | Script size |
|----------|--------------------------|---------------|---------------|
| v2 | `2a52d3f7be80f0e163a2fbd4fa36703e03a0d5a8139a3828e3156c335be59211#0` | 10771553 | 5737 bytes (PlutusV2) |
| v3 | `b3e3b7acef46f70ef511e7ea91c231a6e090a8a9790837a2b83396cf499203b3#0` | (see note) | 5003 bytes (PlutusV2) |

**Note**: The reference scripts are stored *inside* the script addresses themselves (v2 stores the ref at `addr1zxfj9...`, v3 in a variant with a different staking credential but the same payment hash `1af84a9e...`).

**Confirmed via tx_size**: The `add_stake` (1294 bytes) and `consume_rewards` (1363 bytes) txs are too small to include the inline scripts (5737 and 5003 bytes). Therefore they do use reference scripts. The Koios API does not show them in `reference_inputs` (indexing bug).

---

## THERE ARE NO NFTs! — Critical finding

**CONFIRMED**: The credential NFTs mechanism (tracker token + owner NFT) described in the public GitHub and modeled in `main.tx3` is **NOT implemented in the deployed contract**.

Evidence:
- `assets_minted: []` in ALL analyzed staking txs
- The `stake` tx (c49157f3...) has **no Plutus execution** (no collateral) — it is just a UTxO send to the script address
- The `add_stake` and `withdraw_stake` txs execute Plutus (they have collateral) but with no minting/burning
- The deployed contract does NOT have a `mint_policy_id` parameter separate from the script hash

---

## Example transactions per operation

### `stake` — First staking of the protocol

**TX**: `c49157f3896e6396aea334abb21156cf300bc388da9c86ed20a701185516f395`
**Block**: 10762931 (Aug 2024) — v1 contract
**No Plutus execution!** — tx_size 976B, no collateral, `plutus_contracts: []`

```
INPUTS:
  [wallet] 160,644,244 lovelace
  [wallet] 5,448,879 lovelace + 42,850,000 STRIKE

OUTPUTS:
  [wallet] 1,000,000 lovelace  (fee to the batcher?)
  [wallet] 161,867,638 lovelace + 32,850,000 STRIKE  (change)
  [SCRIPT v1] 3,000,000 lovelace + 10,000,000 STRIKE  ← staking UTxO with inline datum
```

**Pattern**: plain UTxO lock, no validator, no redeemer, no NFTs.

---

### `add_stake` — Adds STRIKE to an existing position

**TX**: `334644eca2c585c2cedc630fda259ab1c99b4db49ede39cabc5926527a8c7e76`
**Block**: 13028149 (Mar 2026) — v2 contract
**Executes Plutus** (collateral present), tx_size 1294B → uses reference script

```
INPUTS (Koios incomplete — the script input is missing):
  [wallet] 11,412,898 lovelace + 2,558,636 STRIKE
  [wallet] 9,948,452 lovelace
  [SCRIPT v2] (existing input, not shown by Koios)

OUTPUTS:
  [addr1q8x4...] 6,000,000 lovelace  (batcher/fee address, with datum)
  [wallet]       3,454,877 lovelace  (change)
  [wallet]       5,184,376 lovelace + 128,636 STRIKE  (change)
  [SCRIPT v2]    2,025,192 lovelace + 2,430,000 STRIKE  ← updated staking UTxO
  [SCRIPT v3]    4,349,622 lovelace  (payment to v3 contract/treasury?)
```

---

### `withdraw_stake` — Full withdrawal

**TX**: `a7b53aebec8110b88862de53cf44862bf7064997ca8dd91885045f262e50acfc`
**Block**: 13028259 (Mar 2026) — v2 contract
**Executes Plutus**, similar tx_size → uses reference script

```
INPUTS:
  [wallet]    22,807,219 lovelace + 128,636 STRIKE
  [SCRIPT v2] 4,770,257 lovelace + 3,694,342 STRIKE  ← staking UTxO

OUTPUTS:
  [addr1q8x4...] 6,000,000 lovelace + 147,773 STRIKE  (batcher fee in STRIKE)
  [wallet]       6,000,000 lovelace + 3,546,569 STRIKE
  [wallet]       3,509,593 lovelace + 128,636 STRIKE
  [SCRIPT v2]    11,749,354 lovelace  (residual ADA to the contract)
```

STRIKE total: 3,694,342 + 128,636 = 3,822,978 IN → 147,773 + 3,546,569 + 128,636 = 3,822,978 OUT ✓

---

### `consume_rewards` — Claim rewards without withdrawing stake

**TX**: `71746ba6e072ec3cce487276bbec1005d6736475bdbb917406a341f4860fbd50`
**Block**: 13002691 (Mar 2026) — v3 contract
**Executes Plutus**, tx_size 1363B → uses reference script

```
INPUTS:
  [wallet]    30,000,000 × 3 + 10,000,000 lovelace  (ADA from the batcher for rewards)
  [SCRIPT v3] 2,000,000 lovelace + 121,738,565 STRIKE

OUTPUTS:
  [wallet] 6,000,000 lovelace + 116,550 STRIKE  (batcher fee)
  [wallet] 6,000,000 lovelace + 2,797,202 STRIKE  (rewards to the user)
  [wallet] 82,597,686 lovelace  (ADA returned to the batcher)
  [wallet] 5,035,000 lovelace  (ADA returned)
  [SCRIPT v3] 2,000,000 lovelace + 118,824,813 STRIKE  ← updated position
```

STRIKE: 121,738,565 → 118,824,813 + 2,913,752 (rewards) ✓

---

## Redeemer structure (deployed contract)

### v2 — Spend redeemers

```
Constr(1, [Constr(2, [])])  — 1765 occurrences: main operation (STRIKE route v2→v3)
Constr(1, [Constr(3, [])])  — 2 occurrences: ADA deposit to the pool
Constr(1, [Constr(5, [])])  — 27 occurrences: bulk ADA withdrawal/distribution
```

### v2 — Mint redeemer

```
Constr(0, [])  — 1 occurrence: initial pool creation
```

### v3 — Spend redeemers

```
Constr(0, [Int, Int, Int])  — operation with amount and index parameters
Examples: Constr(0,[120000000,0,0]), Constr(0,[12000000000,1,0])
```

**Note**: The redeemers of the deployed contract are completely different from those of the `main.tx3` based on the public GitHub.

---

## Actual protocol architecture

```
┌─────────────────────────────────────────────────────┐
│                 Strike Finance DEX/Staking           │
│                                                     │
│   v3 (1af84a9e...)          v2 (93229804...)        │
│   addr1zyd0...              addr1zxfj9...           │
│                                                     │
│   - Holds STRIKE stakes     - ADA rewards pool      │
│   - Swap/DEX orders         - Manages ADA distrib.  │
│   - consume_rewards op      - Routes STRIKE to v3   │
│   - 344 active UTxOs        - Multiple 200M ADA UTxOs│
│                                                     │
│   Ref script: b3e3b7ac#0   Ref script: 2a52d3f7#0  │
└─────────────────────────────────────────────────────┘
                        ↓
              Batcher address: addr1q8x4rlq...
              (processes txs, charges fee in STRIKE+ADA)
```

The protocol uses a **batcher model**:
- Users do not interact directly with the script
- A batcher collects requests and processes them in a batch
- The batcher charges a fee in STRIKE (e.g.: 147,773 STRIKE per withdraw)

---

## Actual datum structure (deployed)

The actual on-chain datum does NOT match the `StakingDatum` from the public GitHub.

### Public GitHub (`types.ak`):
```
type StakingDatum {
  owner_address_hash: Hash<Blake2b_224, VerificationKey>,  // 28 bytes
  staked_at: Int,                                          // POSIX ms
  mint_policy_id: PolicyId,                               // 28 bytes
}
```

### Actual on-chain datum (CBOR decoded):
```json
{
  "constructor": 0,
  "fields": [
    {
      // Field 0: full owner Address (not just hash)
      "constructor": 0,
      "fields": [
        {"constructor": 1, "fields": [{"bytes": "<owner_pkh_28bytes>"}]},
        {"constructor": 0, "fields": [{"constructor": 0, "fields": [{"constructor": 0, "fields": [{"bytes": "<stake_pkh>"}]}]}]}
      ]
    },
    {"bytes": "f13ac4d66b3ee19a6aa0f2a22298737bd907cc95121662fc971b5275"},  // staking_policy_id
    {"bytes": "535452494b45"},  // staking_asset_name "STRIKE"
    {"int": 327868852},         // staked_amount
    {"bytes": ""},              // unknown field 1
    {"bytes": ""},              // unknown field 2
    {"int": 930177177},         // staked_at / last_rewarded timestamp
    {"constructor": 1, "fields": []},  // None
    {"constructor": 0, "fields": [     // tracking rewards
      {"constructor": 0, "fields": [{"bytes": "00"}]},
      {"int": 0}
    ]}
  ]
}
```

---

## Discrepancy: tx3 vs deployed contract

| Aspect | Current tx3 (public GitHub) | Deployed contract |
|---------|----------------------------|---------------------|
| Credential NFTs | Yes (mint/burn on stake/withdraw) | **NO** — removed |
| `stake` tx | Executes Plutus (ref script) | Only UTxO lock, no Plutus |
| `owner_address_hash` | `Bytes` (28 bytes PKH) | Full `Address` |
| `mint_policy_id` in datum | 3rd datum field | Not present |
| Asset class STRIKE | Only in `env {}` | Embedded in datum |
| Staked amount | Only in outputs | Recorded in datum |
| Redeemer add_stake | `AddStakeOrConsumeStakingRewards {}` | `Constr(1,[Constr(2,[])])` |
| Redeemer withdraw | `WithdrawStake {}` | `Constr(1,[Constr(5,[])])` (to confirm) |
| `spend_script_ref` | Pending | v2: `2a52d3f7...#0`, v3: `b3e3b7ac...#0` |
| `mint_script_ref` | Yes (NFTs) | **Not applicable** |
| Batcher | Not modeled | Yes, charges fee in STRIKE |

---

## V2 vs V3 datum — key differences

The V2 UTxOs with real STRIKE (>1000) have a datum radically different from V3's:

### V3 datum (staking position):
- 9 fields: full owner Address, asset class STRIKE, staked_amount, timestamps, etc.
- The `owner.payment` field = `ScriptCredential(V2_HASH)` → **V2 is the custodian of V3**
- The `owner.staking` field = `PubKeyCredential(5ea48152...)` → protocol staking cred

### V2 datum (proxy/receipt):
```json
{
  "constructor": 0,
  "fields": [{
    "constructor": 0,
    "fields": [
      {"constructor": 0, "fields": [{"bytes": "<tx_hash_32bytes>"}]},
      {"int": <output_index>}
    ]
  }]
}
```
The V2 datum is simply a **UTxO reference** that points to the actual position in V3. V2 acts as a proxy/custodian layer.

### Architectural implication:
- **V3** = actual store of stakes (owner in datum = V2)
- **V2** = proxy/receipt that references the UTxO in V3
- **Batcher** = manages the flow between V2 and V3

---

## Compiled contract parameters (decoded from the script CBOR)

### v2 — Parameters:
```
Param 0: Address PKH(0e0b0ac4...) + stake(c85decf1...)  → Batcher address 1
Param 1: Address PKH(f961b231...) + stake(2025a198...)  → Batcher address 2
Param 2: Address Script(1af84a9e...) + stake(5ea48152...)  → v3 contract address
```

### v3 — Parameters:
```
Param 0: Address PKH(cd51fc17...) + stake(63c28615...)  → Fee address
Param 1: Address PKH(7c2328db...) + stake(8183f129...)  → Authorize address
```

---

## Recommendations for the tx3

### Option A: Keep it based on the public GitHub (current `main.tx3`)
- Cleaner, possibly the future canonical version
- Does not work with the current mainnet
- Appropriate if Strike Finance plans to redeploy with the GitHub code

### Option B: Rewrite for the deployed contract
Required changes:
1. **Remove** the entire `mint`/`burn` block from the 3 txs
2. **Remove** `mint_script_ref` from `env {}`
3. **Remove** `tracker_asset_name` from `env {}`
4. **Change** `StakingDatum` to 9 fields with a full `Address`
5. **Change** redeemers to the actual deployed structure
6. **Simplify** the `stake` tx — it does not need `reference spend_script` or `collateral`
7. **Add** a batcher fee output in `add_stake` and `withdraw_stake`
8. **Update** `spend_script_ref` in env:
   - v2: `spend_script_ref = 2a52d3f7be80f0e163a2fbd4fa36703e03a0d5a8139a3828e3156c335be59211#0`
   - v3: `spend_script_ref = b3e3b7acef46f70ef511e7ea91c231a6e090a8a9790837a2b83396cf499203b3#0`

---

## Pending

- [ ] Confirm exactly which redeemer `withdraw_stake` uses in v2 (inner=5 likely)
- [ ] Understand the exact relationship between v2 and v3 (which is the main staking?)
- [ ] Confirm whether the Mar 2026 txs (`add_stake`/`withdraw_stake`) use v2 or v3
- [ ] Decode the full datum of an active UTxO in v3 to confirm the 9-field structure
- [ ] Understand the exact role of the batcher address (`addr1q8x4...`)
