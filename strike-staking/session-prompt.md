# Project status — Strike Finance Staking tx3

## Context

Project: `api-layer-tx3-protocols` — Rust/Axum JSON-RPC server that loads `.tii` from `protocols/` and exposes them as JSON-RPC methods.

The `strike-staking` protocol has already been researched on-chain and rewritten. The tx3 compiles and the `.tii` is deployed.

---

## Current status — ALL DONE

### On-chain research ✅

See `strike-staking-research.md` for the full analysis. Executive summary:

**No NFTs** — The public GitHub's credential NFTs mechanism does NOT exist in the deployed contract.

**Actual architecture (two layers)**:
- **V2Script** (`932298043...`, `addr1zxfj9xqyxcut253g30gl5tjaf9qgawp7je87e7zvduhptg275jq4yvpskgayj55xegdp30g5rfynax66r8vgn9fldndsgv4y9t`) — proxy/custodian, the user interacts here
- **V3Script** (`1af84a9e...`, `addr1zyd0sj57d9lpu7cy9g9qdurpazqc9l4eaxk6j59nd2gkh40vvwe5f7xtt25s5fyftlm468rnjznztvgn9p0gvvr72p5qcl3cq7`) — actual store of staked STRIKE
- **Batcher** (`addr1q8x4rlqhrq4rhqhnkamw3fdqmzqgum79yragg4gptcjpphmrc2rpt0exfch4s47fu32amr45vh9wg053hmcx9k7kkcrq6kxftd`) — processes operations, charges fee in STRIKE

**Confirmed reference scripts**:
- V2: `2a52d3f7be80f0e163a2fbd4fa36703e03a0d5a8139a3828e3156c335be59211#0` (5737 bytes PlutusV2)
- V3: `b3e3b7acef46f70ef511e7ea91c231a6e090a8a9790837a2b83396cf499203b3#0` (5003 bytes PlutusV2)

**V2 redeemers (indexed)**:
- `Constr(1,[Constr(2,[])])` — 1765 occurrences (main operation, batcher route V2→V3)
- `Constr(1,[Constr(5,[])])` — 27 occurrences (withdraw/ADA distribution)
- `Constr(0,[])` for mint — 1 occurrence (initial setup)

**V3 redeemers (indexed)**:
- `Constr(0,[Int, Int, Int])` — operation with amount, type and index

**PENDING**: The exact redeemers of `add_stake` and `withdraw_stake` from the user's perspective (Mar 2026 txs) were not indexed yet. The tx3 uses the best available candidates.

### tx3 rewritten ✅

`strike-staking/main.tx3` — compiles and passes `trix check`. Compiled into `protocols/strike-staking.tii`.

**4 modeled transactions**:

| tx | Script | Plutus | Redeemer (best guess) |
|----|--------|--------|-----------------------|
| `stake` | V2Script | NO (simple UTxO lock) | — |
| `add_stake` | V2Script | YES | `V2Redeemer::Action { op: V2Operation::AddStake }` |
| `withdraw_stake` | V2Script | YES | `V2Redeemer::Action { op: V2Operation::Withdraw }` |
| `consume_rewards` | V3Script | YES | `V3Redeemer::Operation { redeem_amount, redeem_op: 1, redeem_idx: 0 }` |

---

## What's MISSING — next session

### 1. Add profiles to `trix.toml`

The `strike-staking/trix.toml` file has no profiles with the real addresses. Add:

```toml
[profile.mainnet]
V2Script = "addr1zxfj9xqyxcut253g30gl5tjaf9qgawp7je87e7zvduhptg275jq4yvpskgayj55xegdp30g5rfynax66r8vgn9fldndsgv4y9t"
V3Script = "addr1zyd0sj57d9lpu7cy9g9qdurpazqc9l4eaxk6j59nd2gkh40vvwe5f7xtt25s5fyftlm468rnjznztvgn9p0gvvr72p5qcl3cq7"
Batcher  = "addr1q8x4rlqhrq4rhqhnkamw3fdqmzqgum79yragg4gptcjpphmrc2rpt0exfch4s47fu32amr45vh9wg053hmcx9k7kkcrq6kxftd"
staking_policy_id  = "f13ac4d66b3ee19a6aa0f2a22298737bd907cc95121662fc971b5275"
staking_asset_name = "535452494b45"
spend_script_ref   = "2a52d3f7be80f0e163a2fbd4fa36703e03a0d5a8139a3828e3156c335be59211#0"
v3_spend_script_ref = "b3e3b7acef46f70ef511e7ea91c231a6e090a8a9790837a2b83396cf499203b3#0"
```

See the exact profiles syntax in `trix.toml` by reviewing ticketing-2026 as a reference or the trix documentation.

### 2. Verify the add_stake / withdraw_stake redeemers

The Mar 2026 txs were not indexed in Koios when the research was done:
- `add_stake`: `334644eca2c585c2cedc630fda259ab1c99b4db49ede39cabc5926527a8c7e76`
- `withdraw_stake`: `a7b53aebec8110b88862de53cf44862bf7064997ca8dd91885045f262e50acfc`

Check whether they are now indexed:
```bash
curl -s "https://api.koios.rest/api/v1/script_redeemers?_script_hash=932298043638b552288bd1fa2e5d49408eb83e964fecf84c6f2e15a1" | \
  python3 -c "
import sys,json
data=json.load(sys.stdin)
for entry in data:
    for r in entry.get('redeemers',[]):
        if r.get('tx_hash') in [
            '334644eca2c585c2cedc630fda259ab1c99b4db49ede39cabc5926527a8c7e76',
            'a7b53aebec8110b88862de53cf44862bf7064997ca8dd91885045f262e50acfc',
        ]:
            print(r['tx_hash'][:16], r['purpose'], r.get('datum_value'))
"
```

If the redeemers differ from those used in the tx3, update `V2Operation` and the mapping.

### 3. Test against mainnet (optional)

Once the correct profiles are in place, test that the JSON-RPC server brings up the protocol correctly and that the transaction templates are valid.

---

## Relevant files

```
strike-staking/
├── main.tx3                           ← protocol source (EDIT HERE)
├── trix.toml                          ← trix project + network profiles
├── README.md                          ← protocol documentation
├── strike-staking-research.md         ← complete on-chain research
├── tx3-limitations-strike-staking.md  ← known tx3 limitations
├── session-prompt.md                  ← this file
├── invoke-args/                       ← sample args per transaction
├── tests/                             ← test fixtures
└── .tx3/tii/                          ← local build output (generated)
```

---

## Key data for quick reference

| Concept | Value |
|----------|-------|
| STRIKE policy ID | `f13ac4d66b3ee19a6aa0f2a22298737bd907cc95121662fc971b5275` |
| STRIKE asset name (hex) | `535452494b45` |
| V2 script hash | `932298043638b552288bd1fa2e5d49408eb83e964fecf84c6f2e15a1` |
| V3 script hash | `1af84a9e697e1e7b042a0a06f061e88182feb9e9ada950b36a916bd5` |
| V2 reference script UTxO | `2a52d3f7be80f0e163a2fbd4fa36703e03a0d5a8139a3828e3156c335be59211#0` |
| V3 reference script UTxO | `b3e3b7acef46f70ef511e7ea91c231a6e090a8a9790837a2b83396cf499203b3#0` |
| Batcher address | `addr1q8x4rlqhrq4rhqhnkamw3fdqmzqgum79yragg4gptcjpphmrc2rpt0exfch4s47fu32amr45vh9wg053hmcx9k7kkcrq6kxftd` |
| stake tx (example) | `c49157f3896e6396aea334abb21156cf300bc388da9c86ed20a701185516f395` |
| add_stake tx (example) | `334644eca2c585c2cedc630fda259ab1c99b4db49ede39cabc5926527a8c7e76` |
| withdraw_stake tx (example) | `a7b53aebec8110b88862de53cf44862bf7064997ca8dd91885045f262e50acfc` |
| consume_rewards tx (example) | `71746ba6e072ec3cce487276bbec1005d6736475bdbb917406a341f4860fbd50` |

---

## Tools

```bash
# V2 redeemers
curl -s "https://api.koios.rest/api/v1/script_redeemers?_script_hash=932298043638b552288bd1fa2e5d49408eb83e964fecf84c6f2e15a1"

# V3 redeemers
curl -s "https://api.koios.rest/api/v1/script_redeemers?_script_hash=1af84a9e697e1e7b042a0a06f061e88182feb9e9ada950b36a916bd5"

# UTxOs of a tx
curl -s "https://api.koios.rest/api/v1/tx_utxos" \
  -H "Content-Type: application/json" \
  -d '{"_tx_hashes":["<hash>"]}'

# Full info of a tx
curl -s "https://api.koios.rest/api/v1/tx_info" \
  -H "Content-Type: application/json" \
  -d '{"_tx_hashes":["<hash>"]}'

# Compile and deploy
cd strike-staking && trix check && trix build
cp .tx3/tii/main.tii ../protocols/strike-staking.tii
```
