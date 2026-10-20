# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:

>The balance check for this system is MockTBillCustodian.realHoldings() * MockPriceFeed.getPrice() / 1e8 >= TBillToken.totalSupply() * 1e10. This asserts that the dollar value of real-world Treasury bills matches or exceeds the total <br>liabilities represented by tokenized shares. However, because realHoldings() relies on a permissioned state variable, it no longer proves real physical custody, instead only proving accounting alignment on-chain while depending on <br>off-chain reporting honesty.

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:

>A legal structure acts as the fail-safe consensus layer because smart contracts cannot physically seize off-chain assets. If the custodian goes bankrupt, the <br>token entitles the holder to a residual legal claim on underlying assets >shielded within the bankruptcy-remote SPV, requiring slow judicial liquidation rather than instant on-chain execution.

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
>A single trusted reporter introduces three primary failure modes:
>1. **Malicious Exploitation / Key Compromise:** An attacker steals the reporter's private key and inputs a massive false NAV (e.g., pumping it by 100x) to drain the vault via over-valued redemptions. *Mitigation (Technical):* Implement a decentralized multi-oracle architecture (like Chainlink Architecture) requiring a multi-signature consensus threshold, or enforce an on-chain programmatic circuit breaker that rejects any NAV price fluctuation exceeding ±1% per day.
>2. **Operational Inactivity / Liveness Failure:** The reporter goes offline due to server crashes or censorship, freezing the asset pricing during highly volatile macroeconomic periods. *Mitigation (Financial/Technical):* Implement a fallback pricing mechanisim using a secondary independent data feed or allow an automated yield accumulation fallback based on a conservative baseline historical rate if no update is pushed within 30 hours.
>3. **Legal / Administrative Fraud:** The issuer forces the reporter to falsify numbers to mask internal insolvency or bad debts. *Mitigation (Legal):* Mandate automated real-time cryptographic attestation links (Proof of Reserve) tied directly to read-only API feeds of a reputable third-party custodian bank, backed by daily independent auditing liabilities.

<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:
>A stale-but-honest NAV creates a highly profitable **arbitrage backdoor** via latency exploitation, resembling a classic financial toxic flow attack. Because the on-chain contract blindly treats the frozen NAV as the absolute truth, it misprices subscriptions and redemptions relative to the real-world macroeconomic market.
>
> Suppose the true real-world value of the underlying short-dated US Treasuries collapses abruptly due to an unprecedented interest rate spike, but the on-chain NAV remains honestly stuck at $1.00. An informed insider or institutional arbitrageur can immediately execute a subscription-redemption arbitrage attack. They can buy heavily discounted shares off-chain or realize that the token is overvalued on-chain, and immediately call `enqueue()` to dump their tokenized shares into the redemption queue. By doing this, they lock in an artificially high payout rate ($1.00) based on the stale on-chain record. When the queue eventually settles, the vault is forced to liquidate real assets at a loss, socializing the toxic deficit onto honest, long-term ecosystem participants who stayed.


---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
>If the NAV rises between enqueue time and settlement time, the **redeemer loses** the unrealized growth because their cash payout value was locked at the lower historical entry rate. Conversely, the **holders who stayed gain** an outsized benefit because the extra accrued interest generated by the underlying T-Bills during that T+1 settlement window remains inside the pool, increasing the backing value for the remaining circulating supply.
>
> This is generally the **correct allocation of risk** from an institutional liquidity standpoint. When a user chooses to exit the fund, they explicitly relinquish their exposure to the asset's future yields in exchange for prioritized liquidity. If the risk profile were inverted (pricing at settlement time), it would incentivize malicious exit-gaming where users front-run volatile yield adjustments. 
>
> If I were to modify it to maximize precision, I would implement a **two-tier processing window charge**: lock the principal asset value at enqueue time to prevent malicious arbitrage, but calculate an isolated fractional accrued-interest rebate paid out to the redeemer upon actual `claim()`, funded explicitly by the specific yield earned during that exact batch's settlement transit time.

<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
>In 2008, Reserve Primary Fund "broke the buck" and completely **froze redemptions**, trapping investor capital and causing systemic panic. In contrast, during the 2023 Silicon Valley Bank crisis, USDC depegged to $0.87 on secondary markets but **kept its redemption channel open** at 1:1, allowing arbitrageurs to burn USDC for cash, which successfully restored the peg. For a tokenized T-Bill fund, "closing the redemption channel" means breaking plank c—users can no longer exit to fiat, turning a liquidity crunch into an existential solvency crisis. If closed, the queue should ideally continue to accept `enqueue()` requests to record exit ordering but halt programmatic `settle()` distributions until funding resume.
>
> The omnipotent `pause()` switch is entirely the **wrong tool** for a liquidity squeeze. By freezing transfers, it destroys secondary market utility, prevents arbitrageurs from supporting the peg, and sparks immense terror. It treats a temporary liquidity mismatch as a total code exploit. 
>
> Instead, I would implement a **graduated redemption gate**: during a liquidity squeeze, transfers and minting remain fully functional, but the redemption queue automatically shifts into a delayed settlement mode, extending settlement from T+1 to T+5 or capping maximum daily cash outflows to 10% of total pool assets.


<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
>The right line for a regulated institutional fund must strike a balance between structural compliance and asset liquidity. Requiring strict whitelist enforcement on both endpoints is legally mandatory for securities compliance (e.g., Regulation S/D or KYC/AML frameworks) to prevent unverified or bad actors from ever holding the token. However, a hardcoded on-chain barrier severely degrades asset velocity.
>The ideal compromise is a two-tier registry structure: the outer token layer allows unrestricted transfers among any addresses that have achieved a baseline, decentralized compliance identification certification (e.g., holding a valid soulbound KYC passport token), while the inner vault restriction strictly controls corporate interactions.
>This compliance boundary should be moved exclusively by a segregated, regulated Compliance Registry Module governed by an independent compliance officer role, rather than the core contract developer or automated financial actors. It should be controlled via a multi-signature arrangement involving the registered Broker-Dealer and Transfer Agent to ensure legal alignment without centralized abuse.
<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
> Pro-Issuer Powers: Handing issuers the absolute ability to freeze and seize is a non-negotiable requirement for real-world compliance. If law enforcement issues a seizure warrant or an address is hacked, the issuer must have tools to confiscate stolen or illegal assets to preserve systemic integrity and avoid massive regulatory penalties.
> Anti-Issuer Powers: Conversely, these absolute censorship options represent a massive centralized backdoor that completely destroys the fundamental trustlessness and neutrality of blockchain architectures, exposing users to arbitrary asset seizure or operational errors.
> The technical collision in Ex4.3 highlights poor architectural separation. To resolve this, I would add a bypass mechanism inside _update that permits permissioned administrative roles (FORCE_BURN_ROLE) to bypass the whitelist checks strictly during a confiscation event.
> Safeguards: I would split these keys across distinct legal actors under an on-chain timelock multi-sig: The setWhitelisted key should be held by the operational compliance desk for rapid emergency freezes, but the FORCE_BURN seizure key must require a 3-of-5 multi-sig held jointly by the Issuer, the independent Custodian, a legal trustee, and an on-chain timelock mechanism, requiring a 48-hour delay before any balance can be permanently destroyed.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | test_Ex2_SharesExistWhileTheCustodianHoldsNothing |
| a | `realHoldings()` moves with no cash moving | test_Ex2_RealHoldingsIsJustANumber |
| b | The claim value floats while the share count stays | test_Ex3_TheClaimFloats_TheShareCountDoesNot |
| b | One `attest` re-prices every holder at once | test_Ex3_OneCallMovesTheWholeBook |
| d | A non-whitelisted address cannot receive shares, even by paying | test_Ex4_NoKyc_NoShares_EvenIfYouPay |
| d | A whitelisted holder cannot send shares to a non-whitelisted address | test_Ex4_TransferChecksBothEndpoints |
| d | Removing an address freezes that holder — and blocks a burn | test_Ex4_FreezeBeatsConfiscation |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
>The scenario most likely to be catastrophic is a "Malicious / Compromised Oracle NAV Injection", which completely breaks Plank B (Attestation).
>If an attacker compromises or bribes the account holding the REPORTER_ROLE, they can call attest() and artificially inflate the NAV from $1.00 to $100.00. Even though the physical assets at the custodian (Plank A) have not changed, the entire on-chain architecture instantly trusts this new number. The attacker can then immediately call enqueue() to dump a small number of shares into the redemption queue, locking in an enormous payout claim value. When the SPV blindly processes the settlement (Plank C), the attacker will drain 100% of the actual underlying cash reserves of the vault, leaving the fund totally bankrupt and rendering the remaining honest holders' tokens completely unbacked and worthless.