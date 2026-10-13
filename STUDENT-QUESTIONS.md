# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

> Your answer: The main risk is that a minter can destroy somebody else's tokens without their permission, because `burn(from, amount)` only checks the caller's role. It never checks whether the owner agreed. Ex4 shows this by making a call as the vault and burning Alice's balance. However, a normal user cannot impersonate the vault on-chain, and the current `redeem()` only burns the caller's tokens. The test proves the role has too much authority, rather than proving anyone can steal immediately.I would remove the role-based arbitrary burn function from `SimpleStablecoin`. I would add a `burnFrom(owner, amount)` function that spends the owner's allowance before burning. In `Vault.redeem()`, the vault would call `stable.burnFrom(msg.sender, amount)`, then return the same amount of collateral to that caller. Alice would first approve the vault to burn only the amount she wants to redeem.This means minting permission no longer gives burning permission over everybody. There is still risks from excessive approvals, so exact allowances are better for this lab. I would also test that redemption without approval fails and leaves both balances unchanged.

<br><br><br>

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

> Your answer:The main improvement is to split the three roles, because one stolen key should not control minting, emergency actions, and permission changes together. In the current constructor, one address receives everything. That makes setup easy, but an attacker who controls that address can give themselves more power even after another role is removed. I would give `DEFAULT_ADMIN_ROLE` to a multisignature wallet controlled by several independent people. Important role changes should go through a delay, so users and monitoring systems can see them before they become active. This delay needs to be enforced in the actual administration path, not just written as a team rule. I would give `MINTER_ROLE` only to approved vault contracts whose minting paths check collateral. After setup, the deployment wallet should lose its own minting role. An emergency security multisig would hold `PAUSER_ROLE`, so it can react faster when something goes wrong. Unpausing could require a separate recovery approval rather than the same quick emergency process. These roles needs different responsibilities. Splitting them reduces one-key failure, although it does not remove bugs in the vault or collusion between the signers.

<br><br><br>

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

> Your answer:The main change is to block transfers and new minting while still allowing authorized burns, because redemption needs to burn sUSD before collateral can be returned. The current `whenNotPaused` modifier on `_update()` blocks every balance change, so it also blocks users from getting their collateral back. I would replace that blanket modifier with a conditional check inside `_update()`. A normal transfer has nonzero `from` and `to` addresses. Minting has `from == address(0)`, while burning has `to == address(0)`. During a pause, I would reject transfers and minting but allow burns to continue through the normal permission checks. Allowing burns here must not mean anyone can burn someone else's balance; the external burn function still needs authorization. With the current redemption flow, the vault can then burn the caller's sUSD and send collateral back. I would test all three paths separately: transfers fail, deposits fail, and redemption succeeds. I would also check that a failed deposit rolls back the collateral transfer. This design helps when the problem concerns trading or minting. If redemption itself is being exploited, a separate emergency redemption control may still be necessary.

<br><br><br>

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:The main difference is that accessible redemption gives holders a way back to the promised value, while freezing it removes that exit. A freeze may stop immediate withdrawals and forced asset sales, but people can become more worried because they do not know when they will get their money back. In 2008, the Reserve Primary Fund reported a value of $0.97 per share and suspended redemption payments. The SEC later allowed a broader suspension. This was more serious than a normal short processing delay. [Source: SEC](https://www.sec.gov/divisions/investment/guidance/reservefundmmffaq.htm).

> The USDC comparison needs a small correction: redemption was not continuously available without interruption. Circle reported banking-related delays and said it had cleared substantially all minting and redemption backlogs by March 15, 2023. [Source: Circle](https://www.circle.com/blog/march-15-update-on-usdc-operations?hss_channel=tw-2151686839).

> The reason a working redemption channel helps is simple. Someone who can buy a coin below $1 and redeem it for $1 has a reason to buy, supporting its market price. However, this depends on reserves being available and redemption being credible. An open channel cannot repair missing assets by itself, and it can let early redeemers leave losses for later holders if the system is insolvent.

<br><br><br>

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer: The main two causes are missing economic backing and blocked access to redemption, because a coin needs both enough assets and a usable way to exchange tokens for them. These problems can look different in the same invariant.

> First, unauthorized or excessive minting can create more sUSD than the vault holds in collateral. Then `totalCollateral() >= totalSupply()` becomes false. Our Ex3 run showed this clearly: supply became 1,001,000 sUSD while collateral stayed at 1,000 mUSDC. That proves undercollateralization. It does not directly measure a market depeg, because our local Anvil chain has no trading market setting a price.

> Second, pausing can block redemption even when collateral still equals supply. The invariant passes, but holders cannot use the promised exit. They may accept a lower market price to leave quickly.

> There is also a weakness in counting token units. If the collateral token itself loses dollar value, 1,000 collateral tokens may be worth less than $1,000 even though the numerical check passes. The main reason is that the code counts balances, not actual dollars. A stronger design needs value checks and redemption availability checks as well as the balance invariant.

<br><br><br>

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer: The attacker can take existing collateral by redeeming unbacked tokens, because the vault checks the caller's sUSD balance and available collateral, but does not ask where those sUSD came from. The key step that should stop this is preventing the attacker from receiving `MINTER_ROLE` in the first place.

> The attacker first uses that role to mint 1,000,000 sUSD to their wallet. No mUSDC enters the vault, so supply rises without matching reserves. If the vault holds at least 1,000,000 mUSDC, the attacker can redeem all those tokens: the vault burns them and transfers that collateral to the attacker. Existing holders are left with a shortage of backing.

> However, the question's full redemption is only possible when enough collateral exists. In our Ex3 state, the vault held only 1,000 mUSDC. Redeeming 1,000,000 sUSD at once would revert with `InsufficientCollateral()`. The attacker could instead redeem 1,000 sUSD and drain the available 1,000 mUSDC, leaving the rest of their tokens unbacked.

> I would restrict minting to collateral-checking vaults and protect role administration with multiple signers. Revoking a compromised role stops more minting, but it does not remove tokens already created.

<br><br><br>

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer: The invariant would need to compare verified asset value with token liabilities, because a Treasury holding cannot be measured reliably just by reading an ERC-20 balance. A token can represent a claim on Treasuries, but the token balance alone does not prove that the securities exist, are unencumbered, and can be sold for the expected amount.

> For this design, I would use the conservative dollar value of verified reserves, minus liabilities and expected sale costs, and compare that with outstanding sUSD. I would also keep a separate liquidity check for how much cash can actually be paid out soon. Enough total assets does not always mean enough cash today.

> The new problems include custody records, price changes, settlement delays, and stale valuation data. Interest also needs accounting: the system must say whether earned interest belongs to token holders, the issuer, or a reserve buffer. A custodian and independent checks would help establish what is held, while an oracle would bring the relevant value on-chain.

> These checks still depends on information outside the blockchain. If the custodian or price source reports incorrectly, the smart contract can calculate a perfectly consistent but misleading number.

<br><br><br>

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer: The vault would hold a legally defined claim linked to the building, because the physical building cannot be transferred into a smart contract. For a conceptual design, I would use a dedicated company or trust to own the property, with tokens representing specified rights in that structure. The exact arrangement would depend on local law and would need legal review.

> The important part is deciding what the token actually gives its holder. It could represent shares, a debt claim, or rights to certain income. Those are different promises, and simply minting an ERC-20 does not automatically transfer legal title to the land.

> The system would need people outside the blockchain: legal advisers to document the rights, a property manager to handle rent and maintenance, an independent valuer, and someone responsible for checking accounts and any mortgage claims. An oracle could report valuations, but it cannot guarantee a buyer will pay that price.

> Redemption would also need a different plan. A building may take months to sell and cannot easily be divided to pay each withdrawing user. Cash reserves, withdrawal queues, and clear loss-allocation rules would therefore matter. The main difficulty is turning a slow physical asset into a credible redemption promise.

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts | `test_Ex4_Mint_RevertsForNonMinter` |
| Transfers revert while paused | `test_Ex4_Pause_BlocksTransfers` |
| **Redemption** reverts while paused | `test_Ex4_Pause_BlocksRedeem` |
| An attacker cannot burn someone else's balance | `test_Ex4_AttackerCannotBurnOthersBalance` |
| ...but the vault holding `MINTER_ROLE` can | `test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance` |

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

> Your answer: The extra scenario I would prioritize is a compromised minter draining collateral, because our Ex3 experiment already showed how minting permission can create unbacked sUSD. I would treat this as a high-priority threat for this design, rather than claim we have statistics proving it is the most common attack.

> My proposed test name is `test_CompromisedMinter_DrainsExistingCollateral`. First, Alice would deposit 1,000 mUSDC normally and receive 1,000 sUSD. Next, the admin would grant the attacker `MINTER_ROLE`, simulating a compromised permission decision. The attacker would mint 1,000 sUSD without depositing anything and redeem those tokens against Alice's collateral.

> The final assertions would check that the attacker received 1,000 mUSDC, the vault's collateral became zero, and Alice still held 1,000 sUSD. I would also check that total supply remained 1,000 sUSD, making the missing backing visible. This shows the actual loss route, rather than only showing that a mint call succeeded.

>  This is a proposed additional test, not one already implemented in the file. Its expected result demonstrates a design risk that needs stronger minting controls; passing that demonstration would not mean the system is secure.
