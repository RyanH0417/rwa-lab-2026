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
The equivalent invariant is MockTBillCustodian.realHoldings() == TBillToken.totalSupply(), this asserts that the custodian's reported holdings match the outstanding token supply. However, unlike Lab 1, it does not prove that the underlying assets actually exist. Since realHoldings() is a permissioned, manually updated value rather than an independently verifiable on-chain ERC-20 balance, the invariant only establishes accounting consistency between two on-chain records. It cannot guarantee actual collateralization without trusting the custodian or relying on independent off-chain verification.
<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:
A legal structure is essential because smart contracts can enforce on-chain rules but cannot guarantee ownership or protection of real-world assets. The SPV legally separates the underlying T-Bills from the issuer, helping protect them from the issuer's bankruptcy. If the custodian goes bankrupt, token holders should have an enforceable claim to the underlying assets or redemption proceeds, depending on the legal agreements. However, the token itself does not guarantee direct ownership or immediate recovery. These rights depend on the legal structure, not the blockchain alone.
<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
A single trusted reporter introduces several risks, including malicious NAV manipulation, private key compromise, human error, and outdated price reporting. These failures could lead to incorrect subscription prices, unfair redemption payouts, or misleading invariants. To mitigate these risks, the system could require independent multi-party approval to prevent manipulation, use hardware-secured keys to reduce key theft, implement sanity checks to catch reporting errors, and enforce freshness checks to reject stale NAV data. However, these measures cannot completely eliminate the reliance on trusted off-chain information.
<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:
A stale-but-honest NAV can be exploited when an attacker knows the true asset value has changed but the on-chain price remains frozen. If the true NAV has increased, the attacker can subscribe at the outdated lower price and receive more tokens than they should. If the true NAV has decreased, they can redeem at the outdated higher price and receive more USDC than their shares are worth. In both cases, the attacker profits from the information gap at the expense of other investors or the protocol. This can be mitigated by checking updatedAt and rejecting subscriptions or redemptions when the NAV is stale.
<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
This allocation of risk may be unfair because the redeemer is still exposed to settlement delays but cannot benefit from price appreciation. A better design would calculate the redemption payout using the NAV at settlement time, allowing the redeemer to receive the updated value. Alternatively, the protocol could clearly define when economic ownership transfers and ensure that both gains and losses are allocated consistently.
<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
In 2008, the Reserve Primary Fund broke the buck and suspended redemptions, eventually liquidating its assets and distributing proceeds to investors. During USDC's 2023 depeg, redemptions were temporarily constrained by banking availability, but Circle maintained its commitment to 1:1 redemption and resumed normal operations. For a tokenized T-Bill fund, closing the redemption channel should mean temporarily stopping new redemption requests or payouts while preserving existing queue entries and their claims. The lab's `pause()` is too broad because it freezes transfers, minting, and redemption simultaneously, even when only redemption liquidity is insufficient. A better design would provide separate controls for subscriptions, transfers, new redemption requests, and settlement, allowing the protocol to manage liquidity without unnecessarily freezing the entire system.
<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
A regulated fund should distinguish between unverified users and genuinely prohibited addresses rather than treating both identically. A tiered whitelist could assign different permissions based on KYC status and regulatory risk, while separate transfer restrictions could allow limited actions such as compliant redemptions without permitting unrestricted trading. Compliance officers should manage routine KYC approvals, but sensitive actions such as freezing assets or overriding restrictions should require multisig authorization and an auditable process. This approach maintains regulatory compliance while reducing unnecessary restrictions and centralized control.
<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
Giving the issuer freezing and burning powers helps a regulated fund comply with sanctions, court orders, and fraud investigations. However, these powers also create risks of censorship, wrongful asset seizure, and abuse by compromised or malicious administrators. The current design also creates a conflict because freezing an address prevents its balance from being burned. I would separate compliance freezing from authorized seizure, allowing forced burns only through a narrowly controlled legal process. Routine whitelist changes should be managed by compliance officers, while forced burns should require multisig approval involving independent legal or compliance representatives. All sensitive actions should be logged on-chain, with clear justification, oversight, and an appeal process for affected users.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing |test_Ex2_SharesExistWhileTheCustodianHoldsNothing |
| a | `realHoldings()` moves with no cash moving |test_Ex2_RealHoldingsIsJustANumber |
| b | The claim value floats while the share count stays |test_Ex3_TheClaimFloats_TheShareCountDoesNot |
| b | One `attest` re-prices every holder at once |test_Ex3_OneCallMovesTheWholeBook |
| d | A non-whitelisted address cannot receive shares, even by paying |test_Ex4_NoKyc_NoShares_EvenIfYouPay |
| d | A whitelisted holder cannot send shares to a non-whitelisted address |test_Ex4_TransferChecksBothEndpoints |
| d | Removing an address freezes that holder — and blocks a burn |test_Ex4_FreezeBeatsConfiscation |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
A likely exploited scenario is a Whitelist Bypass via Secondary Market trading. If a blacklisted or non-whitelisted holder is merely frozen but can still transfer shares through off-chain agreements (OTC) or secondary markets, they can bypass the compliance restriction. Because non-whitelisted addresses cannot receive shares directly on-chain, they might use a whitelisted proxy to receive and immediately redeem the shares for underlying assets. This breaks Plank d (Access Control & Compliance). It exploits the assumption that restricting on-chain transfers guarantees a closed, compliant system, ignoring the fact that economic ownership and redemption rights can be traded off-chain or front-run before a freeze takes effect.