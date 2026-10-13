# Assignment Two: Stablecoin Lab

## 1. Student information

| Item | Details |
|---|---|
| Name | LUI Chi Shing |
| Student ID | 3036326931 |
| Course | COMP7810A Stablecoins, Tokenization, and RWA |
| Assignment | Assignment Two: Stablecoin Lab |

## 2. Repository and test results

**My repository:** [Link](https://github.com/HaroldLui/COMP7810A-Assignment2.git).

The original lab is from [stablecoin-lab-2026](https://github.com/hgwoops/stablecoin-lab-2026). This is the starter repository, not my personal submission link.

### `make test`

![make test screenshot](docs%20%28SUBMISSION%20MATERIALS%29/%22make%20test%22_screenshot.png)

### `make exercise`

![make exercise screenshot](docs%20%28SUBMISSION%20MATERIALS%29/%22make%20exercise%22_screenshot.png)

### `forge test`

![forge test screenshot](docs%20%28SUBMISSION%20MATERIALS%29/%22forge%20test%22_screenshot.png)

Remark: Gave up the optional Ex7 task. The required core and exercise tests passed. Running plain `forge test` also includes Ex7, which still fails.

### Ex3 — Supply exceeds collateral

![Ex3 supply and collateral result](<docs (SUBMISSION MATERIALS)/Ex3_screenshot.png>)

The result shows that supply can become higher than collateral, because an account with `MINTER_ROLE` can mint sUSD without depositing mUSDC. In our Ex3 experiment, supply reached 1,001,000 sUSD while collateral stayed at 1,000 mUSDC. Both tokens use 6 decimals. The permission check works, but giving the permission to an attacker still breaks the backing. This shows missing collateral; the local experiment does not measure a market price.

The threat-model answers are in [STUDENT-QUESTIONS.md](STUDENT-QUESTIONS.md).

## 3. Tier 2 — Sepolia deployment

The three contracts were deployed on Ethereum Sepolia and their source code was verified on Etherscan. The Vault was also checked to reference the correct tokens and hold `MINTER_ROLE`.

| Contract | Sepolia address | Verified source |
|---|---|---|
| MockUSDC | `0x14395E2da1dCB008E96C5E824C29aBaA960F8a91` | [Etherscan](https://sepolia.etherscan.io/address/0x14395E2da1dCB008E96C5E824C29aBaA960F8a91#code) |
| SimpleStablecoin | `0x843FeF9f2B96b86E306e07A0649d58D332B4A42F` | [Etherscan](https://sepolia.etherscan.io/address/0x843FeF9f2B96b86E306e07A0649d58D332B4A42F#code) |
| Vault | `0xDFEaEe83f472843286007c2C8c0888c4F9B83C7b` | [Etherscan](https://sepolia.etherscan.io/address/0xDFEaEe83f472843286007c2C8c0888c4F9B83C7b#code) |

## 4. Architecture and explanation

![Stablecoin architecture](<docs (SUBMISSION MATERIALS)/Architecture_Diagram.png>)

The main idea is user put mUSDC into the vault, then receive the same amount of sUSD. Before deposit, the user calls `approve` on MockUSDC to let the Vault spend their collateral. The Vault then transfers mUSDC from the user to itself and calls `mint` on SimpleStablecoin. MockUSDC records the collateral balance held at the Vault's address.

When user wants the collateral back, they call `redeem`. The Vault burns their sUSD first, then sends mUSDC back. No sUSD approval is needed in this version, because the Vault has permission to burn. Both tokens use 6 decimals, so the amount is matched 1:1.

The main risk is the admin still has minting permission and can grant it to other accounts. A minter can create sUSD without putting collateral in, like what we tested in Ex3. The backing requirement is `totalCollateral() >= totalSupply()`, but direct privileged minting can break it. Also, when sUSD is paused, user cannot redeem even if the Vault still has enough collateral.

This diagram shows the basic 1:1 vault. Ex5 uses a separate overcollateralized vault with mock WETH and a price feed. That system requires at least 150% collateral for minting and withdrawals, and allows liquidation below 120%.
