# Lab 2 — How to Tokenize an RWA (Tokenized T-Bills)

In Lab 1 the collateral was **itself a token**. That is why the whole thing could prove itself:
`totalCollateral() == totalSupply()` was a statement the chain could check on its own, and nothing
outside the code had to be trusted.

This lab removes that assumption. The asset is now a **short-dated US Treasury bill**, sitting at a
custodian. What is on-chain is not the asset — it is a **claim on the asset**. The subject of this
lab is the machinery that connects the two:

> **The bridge.** Custody, attestation, physical redemption, and permissioning — the four things
> that have to exist before an off-chain asset can become an on-chain token, and the four places
> where such a token breaks.

Every exercise below is built around a plank of the bridge. If you find yourself only reading NAV
numbers and yields, you have drifted back into Lab 1.

---

## 1. Setup

Pick the track that matches your machine. **Everything after this section is identical for everyone.**

| You are on | Track |
|---|---|
| macOS / Linux | **A** — install Foundry locally |
| Windows | **B** — Codespaces, nothing installed on your machine |

> `lib/` (`forge-std`, `openzeppelin-contracts`) **ships inside this repository**.
> Neither track needs `git clone --recursive`, and neither needs `make setup` —
> a plain clone compiles as-is.

### Track A — macOS / Linux

**Step 1. Install Foundry**

First check whether you already have it:

```bash
forge --version
```

If that prints a version, skip to Step 2. Otherwise install it with the script in this repo (**do not use `foundryup`**):

```bash
bash scripts/install-foundry-cn.sh
```

The script detects your platform, tries GitHub directly, and falls back to the `gh-proxy.com`
mirror. Add the line it prints to `~/.zshrc` or `~/.bashrc`, then reopen your terminal:

```bash
export PATH="$PATH:$HOME/.foundry/bin"
```

> ⚠️ **Why not `foundryup`**: its download goes through GitHub's CDN, which times out reliably from
> mainland China. The mirror does not.

**Step 2. Get the code**

```bash
git clone https://github.com/hgwoops/rwa-lab-2026 && cd rwa-lab-2026
```

Then go to **Verify** below.

### Track B — Windows (Codespaces)

Foundry has **no native Windows binary**, so the straightforward path is to run the lab on
GitHub's servers from your browser. Nothing is installed locally, which also makes it the only
option on locked-down corporate machines.

1. Open <https://github.com/hgwoops/rwa-lab-2026>
2. Green **`Code`** button → **`Codespaces`** tab → **`Create codespace on main`**
3. **The first launch takes 2–5 minutes**: it builds the environment from `.devcontainer/`,
   then automatically runs `make doctor` — you should see a row of `✓` in the terminal
4. In the terminal at the bottom, run:

```bash
make test
```

`7 passed; 0 failed` means you are set.

The free tier is 120 core-hours/month — far more than this lab needs. **Stop it when you are done**
at <https://github.com/codespaces>; a running Codespace keeps burning quota.

### Verify (both tracks)

```bash
make doctor
```

In about 30 seconds this checks the toolchain, the dependencies, and runs the core tests. Anything
marked `✗` comes with the exact command to fix it. All green means you can start; if you cannot fix
it, paste the whole `make doctor` output to the TA.

> New to Foundry? Read **`FOUNDRY-101.md`** before you touch the tests: how a test is shaped, the
> cheatcodes you will need, the command band, and where to look when it breaks.

---

## 2. Quick start

```bash
make doctor        # environment check — all green before you go on
make test          # the checkpoint: core tests, should be all green (7 passed)
make exercise      # the hands-on tasks; red right now — the red ones ARE your task list
make anvil         # terminal A: start a local chain
make deploy-anvil  # terminal B: deploy to it
```

Deployment prints the admin plus seven contracts. Export the ones you need (`$USDC`, `$COMPLIANCE`,
`$FEED`, `$TBILL`, `$CUSTODIAN`, `$VAULT`, `$QUEUE`); the run-it-by-hand commands are in
`EXERCISES.md` Ex1.

**Your task list is `EXERCISES.md`** — every exercise's goal, acceptance command, and where to look.

---

## 3. What the contracts do

The bridge has four planks. Each one is a contract or a role in this repo.

| File | Role | Plank |
|---|---|---|
| `src/MockUSDC.sol` | **Cash.** 6 decimals, with a test faucet. The thing that flows across the bridge | — |
| `src/ComplianceRegistry.sol` | **The guest list.** `setWhitelisted()` / `isWhitelisted()`, gated by `COMPLIANCE_ROLE` | (d) admission |
| `src/TBillToken.sol` | **The claim.** ERC-20, **18 decimals**, mint/burn gated, and every balance change must pass both endpoints' whitelist | (d) admission |
| `src/TBillVault.sol` | **The on-ramp and the books.** `subscribe()` prices shares at the attested NAV; `attest()` is the reporter's only way to move it | (b) attestation |
| `src/exercises/MockTBillCustodian.sol` | **The custodian / SPV.** Answers exactly one question: `realHoldings()` — how much is actually there? | (a) custody |
| `src/exercises/MockPriceFeed.sol` | **The NAV oracle.** 8 decimals; `1.00e8` is par | (b) attestation |
| `src/exercises/RedemptionQueue.sol` | **T+1.** Shares in, a queue ticket out, cash later — **your TODOs, Ex5** | (c) redemption |

### The two invariants you will defend (Ex6)

```
escrow conservation   tBill.balanceOf(queue) == queue.pendingShares()
settlement solvency   usdc.balanceOf(queue) >= queue.totalClaimable()
```

The first says the queue never loses a share and never invents one. The second says it never owes
settled cash it does not hold. Neither is something the chain can verify against the off-chain
asset — that is the point. The three decimal scales (18 + 8 − 6) are in `EXERCISES.md` Ex5; the
third relation that *looks* like an invariant and is not is Ex6.

`script/Deploy.s.sol` wires the whole system together, and the wiring at the bottom **is** part of
the lesson: the vault needs `MINTER_ROLE`; the queue needs `MINTER_ROLE` **and** must be
whitelisted (burning its escrowed shares is a balance change); the vault must be told where the
queue is; the custodian must recognize the vault. Skip any one and the loop reverts.

---

## 4. Homework

**Tier 1 (required)** — the exercises **Ex0–Ex6** in `EXERCISES.md`, and the discussion questions in
`STUDENT-QUESTIONS.md`. Ex1 is a warm-up and is **not graded**. A local Anvil is enough: Tier 1 does
**not** require a testnet. The acceptance command for every exercise is `make exercise` — green when
you are done.

```test result
root@codespaces-59b61e:/workspaces/rwa-lab-2026# make exercise
forge test --match-path 'test/exercises/*.t.sol' -vv
[⠒] Compiling...
[⠆] Compiling 1 files with Solc 0.8.24
[⠰] Solc 0.8.24 finished in 2.12s
Compiler run successful!

Ran 7 tests for test/exercises/01_BridgeTasks.t.sol:BridgeTasksTest
[PASS] test_Ex2_RealHoldingsIsJustANumber() (gas: 72667)
[PASS] test_Ex2_SharesExistWhileTheCustodianHoldsNothing() (gas: 262669)
[PASS] test_Ex3_OneCallMovesTheWholeBook() (gas: 582039)
Logs:
  aliceclaimvalue:  1000000000
  bobclaimvalue:  1000000000
  aliceclaimvalue2:  50000000000
  bobclaimvalue2:  50000000000

[PASS] test_Ex3_TheClaimFloats_TheShareCountDoesNot() (gas: 334423)
[PASS] test_Ex4_FreezeBeatsConfiscation() (gas: 377183)
[PASS] test_Ex4_NoKyc_NoShares_EvenIfYouPay() (gas: 230024)
[PASS] test_Ex4_TransferChecksBothEndpoints() (gas: 315032)
Suite result: ok. 7 passed; 0 failed; 0 skipped; finished in 10.05ms (4.50ms CPU time)

Ran 11 tests for test/exercises/02_QueueTasks.t.sol:QueueTasksTest
[PASS] test_Ex5_1_AssetsAtNav_AtHigherNav() (gas: 11485)
[PASS] test_Ex5_1_AssetsAtNav_AtPar() (gas: 5804)
[PASS] test_Ex5_2_Enqueue_EscrowsSharesAndLocksNav() (gas: 560980)
[PASS] test_Ex5_2_Enqueue_LocksTheRate() (gas: 577245)
[PASS] test_Ex5_3_Settle_NeedsTheVaultToHoldCash() (gas: 1101590)
[PASS] test_Ex5_3_Settle_PartialFillCarriesTheRemainder() (gas: 1140091)
[PASS] test_Ex5_3_Settle_PaysTheWholeQueue() (gas: 1177459)
[PASS] test_Ex5_3_Settle_StopsWhenItCannotCoverTheHead() (gas: 565195)
[PASS] test_Ex5_4_Claim_CannotBeDrawnTwice() (gas: 770713)
[PASS] test_Ex5_4_Claim_PaysOutAndZeroesTheBalance() (gas: 792009)
[PASS] test_Ex5_4_Claim_RevertsWithNothingToClaim() (gas: 34098)
Suite result: ok. 11 passed; 0 failed; 0 skipped; finished in 10.73ms (6.47ms CPU time)

Ran 1 test for test/exercises/03_InvariantTasks.t.sol:InvariantTasksTest
[PASS]
InvariantTasksTest invariants:
[PASS] invariant_EscrowConservation
[PASS] invariant_SettlementSolvency
 InvariantTasksTest invariants (runs: 256, calls: 128000, reverts: 0)

╭---------------+-----------+-------+---------+----------╮
| Contract      | Selector  | Calls | Reverts | Discards |
+========================================================+
| BridgeHandler | attest    | 21312 | 0       | 0        |
|---------------+-----------+-------+---------+----------|
| BridgeHandler | claim     | 21244 | 0       | 0        |
|---------------+-----------+-------+---------+----------|
| BridgeHandler | enqueue   | 21325 | 0       | 0        |
|---------------+-----------+-------+---------+----------|
| BridgeHandler | fundVault | 21369 | 0       | 0        |
|---------------+-----------+-------+---------+----------|
| BridgeHandler | settle    | 21326 | 0       | 0        |
|---------------+-----------+-------+---------+----------|
| BridgeHandler | subscribe | 21424 | 0       | 0        |
╰---------------+-----------+-------+---------+----------╯

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.07s (14.06s CPU time)

Ran 3 test suites in 14.09s (14.09s CPU time): 19 tests passed, 0 failed, 0 skipped (19 total tests)
```

**Tier 2 (bonus)** — deploy to Sepolia and verify the source on Etherscan; put **all seven**
addresses and their Etherscan links in your `README.md`, under a `## Deployment (Sepolia)` heading.
The deploy prints them in this order: `MockUSDC`, `ComplianceRegistry`, `MockPriceFeed`,
`TBillToken`, `MockTBillCustodian`, `TBillVault`, `RedemptionQueue`. The commands are Lab 1's
`README.md` §4, under "Tier 2 (bonus)" — the flow is identical.

```bash
source .env
forge script script/Deploy.s.sol:Deploy \
  --rpc-url $SEPOLIA_RPC_URL \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --private-key $PRIVATE_KEY \
  --gas-estimate-multiplier 2000
```

>MockUSDC: 0x9E4C89bBF2b59ea28bBCFEeC3A7775aeDEFE8981, https://sepolia.etherscan.io/address/0x9E4C89bBF2b59ea28bBCFEeC3A7775aeDEFE8981
>ComplianceRegistry: 0x1a138846A033eBAd0a3Bf9Ea375e198Cd2C922e0, https://sepolia.etherscan.io/address/0x1a138846A033eBAd0a3Bf9Ea375e198Cd2C922e0
>MockPriceFeed: 0x94E362049C2f32B4605F201CEc23a17B7457383a, https://sepolia.etherscan.io/address/0x94E362049C2f32B4605F201CEc23a17B7457383a
>TBillToken: 0x5B2e2d276D7e906D1F42822551D8f5dC91Cc54D5, https://sepolia.etherscan.io/address/0x5B2e2d276D7e906D1F42822551D8f5dC91Cc54D5
>MockTBillCustodian: 0xa77787529fBC1950E62E12a8eD27f5cbCfe181a9, https://sepolia.etherscan.io/address/0xa77787529fBC1950E62E12a8eD27f5cbCfe181a9
>TBillVault: 0x92AE9329aBae11b90585598C32bBfd802c17F910, https://sepolia.etherscan.io/address/0x92AE9329aBae11b90585598C32bBfd802c17F910
>RedemptionQueue: 0x022e264835B857a21103F66a5Fdb8Ff052cA20c0, https://sepolia.etherscan.io/address/0x022e264835B857a21103F66a5Fdb8Ff052cA20c0
>Admin: 0xD4BF270e7056DDa525575a15eDFb961885Bb606c, https://sepolia.etherscan.io/address/0xD4BF270e7056DDa525575a15eDFb961885Bb606c

**Tier 3 (challenge, optional)** — **Ex7**: `make challenge` (`test/challenges/FalseNav.t.sol`). The
reporter's number is the chain's only window onto the asset; show what happens when it lies, and
name the mechanism that would have caught it.

---

## 5. Common problems

**`Ownable` / role errors**
OpenZeppelin v5 has breaking changes, so v4 tutorials from the web will fail. This repo pins
`v5.0.2` — do not upgrade it.

**`subscribe` reverts with `NotWhitelisted`**
The token is permissioned. Whitelist the recipient first — and remember the **queue** must be
whitelisted too, because burning its escrowed shares counts as a balance change.

**`invest` reverts with `AccessControlUnauthorizedAccount`**
The custodian does not recognize the vault. `script/Deploy.s.sol` grants
`custodian.CUSTODIAN_ROLE()` to the vault — if you wired things by hand, you skipped it.

**Amounts are off by an order of magnitude**
`1000e6 = 1_000_000_000` in smallest units (USDC). Shares are 18 decimals and the NAV is 8 — the
conversion is `shares * nav / 1e20`. Do not compute 18-decimal to 18-decimal.

**Addresses change after restarting Anvil**
Anvil starts from a clean state every time, so you must redeploy.

**`make exercise` hangs, or an invariant failure looks strange**
Foundry replays a cached counterexample from `cache/invariant/` on the next run.
To search for a fresh one, `rm -rf cache/invariant` first.

---

## 6. Submission

Submission follows Lab 1: **one zip of the project** on Moodle, built from the project root.

```bash
rm -f .env        # a private key anywhere in the zip scores zero
zip -r submission.zip . -x '.git/*' 'out/*' 'cache/*' 'broadcast/*' '.env' '.DS_Store'
```

**Keep `lib/` inside it** — those vendored dependencies are what let the grader run `make test`
offline. Drop them and the grader has to fetch from GitHub.

What the zip must contain:

1. The code, with `make exercise` all green (the grader re-runs `make test` themselves)
2. Your answers to the discussion questions in `STUDENT-QUESTIONS.md`
3. An architecture diagram in your `README.md` — a photo of a hand drawing is fine, but it must
   show the **four planks** and where cash and shares cross the bridge

### Architecture: the four planks

```mermaid
flowchart LR
    investor[Investor / holder]
    reporter[Authorized NAV reporter]
    spv[SPV / T-Bill seller]
    bills[(Off-chain T-Bills)]

    subgraph admission["ADMISSION — KYC-gated share ownership"]
        registry["ComplianceRegistry<br/>whitelist"]
        token["TBillToken<br/>tBILL shares"]
        registry -->|"checks sender and receiver on mint, transfer, burn"| token
    end

    subgraph attestation["ATTESTATION — reported value"]
        feedInterface["IPriceFeed"]
        feed["MockPriceFeed<br/>8-decimal NAV"]
        feedInterface -. "implemented by" .-> feed
    end

    subgraph custody["CUSTODY — cash buffer and off-chain asset"]
        cash["MockUSDC<br/>6-decimal cash"]
        vault["TBillVault<br/>subscription, NAV pricing,<br/>reserve buffer"]
        custodian["MockTBillCustodian<br/>reported holdings"]
        cash -->|"subscribe: USDC"| vault
        vault -->|"invest: USDC"| custodian
        custodian -. "custody represented off-chain;<br/>mock records a number only" .-> bills
        spv -->|"T+1 sale proceeds:<br/>depositReserves"| vault
    end

    subgraph redemption["REDEMPTION — escrow, FIFO settlement, claim"]
        queue["RedemptionQueue<br/>ticket and claimable USDC"]
    end

    investor -->|"approve + enqueue:<br/>tBILL shares escrowed"| queue
    vault -->|"subscribe: mint tBILL"| token
    reporter -->|"attest(NAV)"| vault
    vault -->|"writes NAV"| feed
    feed -->|"navPerShare used for pricing"| vault
    queue -->|"locks payout at current NAV"| vault
    queue -->|"settle: releaseReserves request"| vault
    vault -->|"USDC reserves"| queue
    queue -->|"settlement burns escrowed shares"| token
    queue -->|"claim: USDC payout"| investor
```

**Flow key:** cash enters the vault on subscription, may be invested with the custodian, and
returns from the SPV after a T+1 sale before the queue can settle and pay a claim. Shares are
minted to admitted holders on subscription, escrowed by the queue on redemption, and burned when
their ticket settles. The NAV reporter updates the feed through the vault; the feed is a reported
number, not proof of custody. `ComplianceRegistry` gates every non-zero share-balance endpoint.

---

## 7. Questions left to you

The discussion prompts live in `STUDENT-QUESTIONS.md` — one per plank, no standard answers, and the
real point of this lab.
