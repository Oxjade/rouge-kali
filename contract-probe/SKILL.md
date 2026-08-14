---
name: contract-probe
description: |
  Passive, read-only structure-mapping skill for smart contract audit pipelines.
  Builds the complete ground-truth map of a codebase — every function/instruction,
  account/storage, PDA/seeds, CPI/external call, state machine, economic flow,
  trust boundary, and guardrail — WITHOUT executing code, mutating state, or
  judging findings. Consumes source (Solidity/Foundry, Solana Rust/Anchor,
  Pinocchio, native solana_program) and emits structure_map.json plus
  accessibility and bypass matrices. Other audit agents query this map instead
  of re-reading raw source. Use when a security task needs "what exists, what is
  reachable, what is bypassable" before analysis begins, or when delegating
  enumeration to a background agent while the main audit runs.
  Not for finding vulnerabilities — for mapping the territory they live in.
license: MIT
version: 1.0.0
---

# Contract Structure Probe

## Mission

Produce the ground-truth structural map of a smart contract codebase. You are READ-ONLY: never execute the code, never send transactions, never mutate state, never judge findings. You answer three questions:

1. **What exists?** — complete inventory, zero assumptions
2. **What is reachable?** — every entry point × every caller state
3. **What is bypassable?** — every guardrail × every known bypass technique

Your output is the single source of truth that all deeper analysis consumes.

## Working Rules

- **Read-only, always.** No execution, no deployment, no state changes, no on-chain writes.
- **Zero assumptions.** "Probably not exposed" is not a reason to skip. If it's in the code, it's cataloged. Every function, every account, every storage slot.
- **Verbatim facts.** Record what the code says. Never infer intent, never fix, never rate. Findings judgment belongs to the audit phase.
- **Persist everything** under `~/rouge-kali-state/<target>/<track>/` — never `/tmp`.

## Phase 1 — Enumerate (What exists?)

### EVM (Solidity/Foundry/Hardhat)

```bash
# Every contract, function, storage variable, event
find <target> -name "*.sol" -type f | sort > structure/contracts.txt
grep -rn "function \|constructor\|modifier " --include="*.sol" <target> | sort > structure/functions.txt
grep -rn "mapping(\|struct \|uint\|address\|bool\|enum " --include="*.sol" <target>/src | sort > structure/storage.txt
grep -rn "event " --include="*.sol" <target> | sort > structure/events.txt

# External calls, proxy patterns, guardrails
grep -rn "\.call{\|\.delegatecall\|\.transfer\|\.send(" --include="*.sol" <target> | sort > structure/external_calls.txt
grep -rn "nonReentrant\|onlyOwner\|whenNotPaused\|require(" --include="*.sol" <target> | sort > structure/guardrails.txt
grep -rn "delegatecall\|UUPS\|TransparentUpgradeableProxy\|ERC1967\|beacon" --include="*.sol" <target> | sort > structure/proxies.txt

# Entry points (deployment scripts, factories)
grep -rn "new \|deploy" --include="*.sol" <target> | sort > structure/entry_points.txt
```

### Solana (Anchor / native / Pinocchio)

```bash
# Instructions, accounts, PDAs, CPIs, authorities
grep -rn "pub fn " --include="*.rs" <target>/programs/ | sort > structure/instructions.txt
grep -rn "derive(Accounts)\|Account<'info\|UncheckedAccount\|AccountInfo" --include="*.rs" <target>/programs/ | sort > structure/accounts.txt
grep -rn "seeds = \|bump\|has_one\|init\b\|signer\|mut " --include="*.rs" <target>/programs/ | sort > structure/constraints.txt
grep -rn "invoke\|invoke_signed\|cpi::" --include="*.rs" <target>/programs/ | sort > structure/cpis.txt
grep -rn "transfer_authority\|authority\|owner" --include="*.rs" <target>/programs/ | sort > structure/authorities.txt
grep -rn "enum.*State\|enum.*Status" --include="*.rs" <target>/programs/ | sort > structure/state_machines.txt
```

## Phase 2 — Accessibility (What is reachable?)

For EVERY entry point, build the accessibility matrix:

| Entry point | Auth state A | Auth state B | ... | Every method |
|---|---|---|---|---|
| `withdraw(user)` | reachable | reachable | ... | all paths tested |
| `admin.pause()` | blocked | reachable | ... | 1 path |

- EVM: every external/public function × caller role (EOA / contract / proxy / owner)
- Solana: every instruction × caller state (authority / non-authority / PDA / token-2022 holder / other program via CPI)

Record: reachable / blocked-by-guardrail / internal-only.

## Phase 3 — Bypass (What is bypassable?)

For EVERY guardrail in the inventory, enumerate known bypass techniques and record tested results. Test = static reasoning over the code (read-only — never exploit for real):

| Guardrail | Technique | Result |
|---|---|---|
| `nonReentrant` | cross-contract reentrancy | bypassable / not |
| `onlyOwner` | delegatecall selfdestruct | bypassable / not |
| signer check | PDA-signer misuse | bypassable / not |
| upgrade authority | authority-hijack paths | bypassable / not |

Known technique catalogs: EVM — reentrancy variants, flash loans, oracle manipulation, proxy storage collision, signature replay, cross-chain replay, EIP-1967 slot attacks; Solana — non-canonical bump, account confusion, type cosplay, CPI authority reuse, token-2022 hook bypass, close-account reuse, reinitialization.

## Output — structure_map.json (MANDATORY)

```json
{
  "track": "evm | solana",
  "framework": "foundry | anchor | pinocchio | native",
  "protocol_type": "lending | amm | ...",
  "entry_points": [],
  "functions": {"name": {"visibility": "", "writes": [], "external_calls": [], "guardrails": []}},
  "storage": {},
  "accounts": {},
  "pda_seeds": {},
  "cpi_graph": [],
  "state_machines": [],
  "economic_flows": [],
  "guardrails": [],
  "upgrade_authority": null,
  "accessibility_matrix": [],
  "bypass_matrix": []
}
```

Write to `~/rouge-kali-state/<target>/contracts/structure_map.json` (EVM) or `~/rouge-kali-state/<target>/solana/structure_map.json` (Solana).

## Handoff

Your map is consumed by:
- **Track B agents** (EVM): 12-agent audit queries the map, not source
- **Track C agents** (Solana): 12-agent audit queries the map, not source
- **Fuzzer generation**: invariants derived from your state machines + economic flows
- **PoC gate**: verified facts only — no re-derivation errors

Report back: path to structure_map.json, inventory counts (functions, accounts, guardrails, bypass results), and a one-line protocol summary. No findings, no severity, no opinions.