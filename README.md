# PoC for Bull.family bug bounty report B-15

## SDK `deal()`: pre-send check() validates payee/amount/expiry but never conditions — server swaps in a self-attested agreement and pays out without the arbiter

**Report B-15** (K-356). SDK under test: `sdk-live.mjs` — the NEW Bull SDK
live 18 Nov 2026, sha256 `220d326f…`, 26,232 bytes
(`https://www.bull.family/sdk/bull.mjs`).

### The bug

The SDK states: *"Safety: the API server is not trusted with your wallet.
Every transaction it returns is checked before it is sent."* The 18 Nov
patch added a pre-send `check()` to `deal()` — but it validates ONLY
payee/asset/amount/expiry. **Conditions are never inspected.** The server
can keep the requested payee and amount and swap the condition set.

A compromised/malicious API server, against a victim who calls `deal()`
for a 500-token agreement payable to a contractor with attest-by-arbiter
as the release condition, returns a `createAgreement` step with the same
payee, asset, amount and expiry — but conditions `[attest-by-payee]`
instead of `[attest-by-arbiter]`. `deal()`'s `check()` passes it; the
payee self-attests and releases the 500 tokens with zero arbiter
involvement. Control, proven on live Agreement bytecode: with the honest
arbiter target, the payee CANNOT attest at all and cannot release — the
swap is exactly what makes the payment unconditional.

**Measured (fresh Anvil, live Agreement v3 bytecode):** agreement #3
created; payee self-attested, released 500 tokens; arbiter involvement:
none.

### Why this is not a duplicate

- **B-07** (report `ce3b4b4f7f`) was "`deal()` verifies NOTHING" — the
  payee swap. The NEW SDK's `check()` was clearly written to close B-07
  (payee/asset/amount/expiry are now verified) and it does close the payee
  swap. What is open now is a different field set — the conditions —
  which the new `check()` never looks at.
- **B-04** (report `7ae78fa73a`) is a different function (`agree()`,
  whose new `check()` DOES verify attest/time_after conditions).
  `deal()`'s `check()` verifies no conditions at all.

### Threat model (stated honestly)

This requires the Bull API server to be compromised or malicious. That is
the SDK's own documented threat model ("the API server is not trusted").
The attacker needs no victim private key and no victim action beyond a
normal `deal()` call. No server compromise is demonstrated; the claim is
the SDK fails its own stated safety goal.

### Reproduce

Local Anvil only (chain 4663), mock malicious API, real NEW SDK verbatim,
live Agreement bytecode etched. No real funds, no mainnet, no Bull
infrastructure. Throwaway local (Anvil default) keys only.

```bash
# terminal 1
anvil --port 8547 --chain-id 4663

# terminal 2
npm install            # installs viem
node run-poc.mjs       # 8 cases; relevant case: "K-356 deal() conditions blind spot"

# forge tests (live Agreement bytecode)
cd chain && forge test # relevant: testK356DealConditionsBlindSpot
```

Expected runner output for the case: `EXPLOITED: true` — SDK sent the
step with NO ERROR, NO WARNING; agreement #3 created; payee attested +
released 500 tokens.

### Files

- `run-poc.mjs` — node runner (mock API + real SDK + fresh Anvil)
- `sdk-live.mjs` — verbatim copy of the NEW live SDK (18 Nov 2026)
- `poc-result.json` — full round-9 results (all 5 PASS cases + negatives)
- `chain/` — Foundry tests on live Agreement/Escrow/Burner bytecode
  (`test/Round9A.t.sol`, `bytecode/`, `src/Mocks.sol`)

### Suggested fix

In `deal()`'s `check()`, compare the decoded `createAgreement`
conditions against the conditions the caller asked for (kinds, targets,
data, values) — the same way `agree()`'s `check()` now verifies
attest/time_after conditions, extended to all kinds.
