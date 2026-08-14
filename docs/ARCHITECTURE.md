# Canton Vault Architecture & Technical Specification

This document provides a comprehensive technical manual for **Canton Vault** (`canton-vault`), detailing mathematical foundations, ledger privacy patterns, institutional compliance safeguards, template definitions, and exhaustive function specifications.

---

## 1. Executive Summary & Design Philosophy

**Canton Vault** is an institutional tokenized asset yield vault built natively for the **Canton Web3 Network** using Daml smart contracts. It integrates with Canton's official token standards (**CIP-0056**, **CIP-0112**, **CIP-0086**) implementing:

1. **Virtual Share Offset Math ($10^3$)**: Neutralizes inflation rounding attacks.
2. **Dual-Signatory UTXO Security**: Protects against un-requested token donation attacks.
3. **Canton Privacy vs. Public Disclosure**: Preserves user privacy on individual holdings while exposing real-time aggregate Net Asset Value (NAV) metrics to public network observers.
4. **Institutional Compliance & Regulator Observers**: Supports real-time SEC/FINRA regulatory auditability via sub-transaction observer privacy without public data leakage, explicit KYC/AML allowlists, sanctions screening, and post-deposit forced legal escrow liquidation.
5. **Balance Sheet Invariant**:
   $$\text{TotalAssets} = \text{IdleAssets} + \sum \text{StrategyAssets}$$

---

## 2. Mathematical Specifications (`CantonVault.Types`)

### 2.1 Virtual Share Offset Math ($10^3$)

To neutralize inflation rounding attacks, the vault maintains a virtual offset exponent $E = 3$ (giving virtual shares offset $10^3 = 1000$).

#### Deposit Conversion (`convertToShares`)
Calculates vault share tokens to mint for a given deposit asset amount:

$$\text{sharesToMint} = \left\lfloor \frac{\text{depositAssets} \times (\text{totalShares} + 10^E)}{\text{totalAssets} + 1} \right\rfloor$$

- **Parameters**:
  - `depositAssets : Decimal`: Amount of underlying tokens deposited.
  - `totalAssets : Decimal`: Current total underlying assets managed by vault.
  - `totalShares : Decimal`: Current total vault share supply.
  - `offsetExp : Int`: Exponent for virtual offset ($3$).
- **Expected Behavior**: Truncates output at Daml `Decimal` fixed precision ($10^{-10}$ atomic units), ensuring rounding always favors the vault over the depositor.

#### Redemption Conversion (`convertToAssets`)
Calculates underlying assets to return for a given share redemption amount:

$$\text{assetsToReturn} = \left\lfloor \frac{\text{redeemShares} \times (\text{totalAssets} + 1)}{\text{totalShares} + 10^E} \right\rfloor$$

- **Parameters**:
  - `redeemShares : Decimal`: Amount of vault share tokens burned.
  - `totalAssets : Decimal`: Current total underlying assets managed by vault.
  - `totalShares : Decimal`: Current total vault share supply.
  - `offsetExp : Int`: Exponent for virtual offset ($3$).
- **Expected Behavior**: Truncates output at Daml `Decimal` fixed precision ($10^{-10}$ atomic units), ensuring rounding always favors the vault over the redeemer. If $\text{redeemShares} = \text{totalShares}$ (100% redemption), all remaining dust assets are returned to cleanly empty the vault.

---

## 3. Institutional Compliance & Regulatory Architecture

### 3.1 Sub-Transaction Regulator Observers
Canton Vault allows financial institutions (DTCC, JPMorgan, Citi, State Street) to fulfill regulatory compliance requirements by designating `regulatorObservers : [Party]` on `VaultConfig` and `VaultState`. Regulator nodes receive 100% real-time, tamper-proof sub-transaction updates for all deposits, yield harvests, strategy allocations, and redemptions without exposing private investor identities to public network nodes.

### 3.2 Permissioned Vaults & Compliance Attestations (`CantonVault.Compliance`)
Permissioned vaults enforce on-chain compliance validation via `ComplianceAttestation` contracts:
- `isKycApproved : Bool`: Subject has completed institutional KYC verification.
- `isAmlCleared : Bool`: Subject has passed Anti-Money Laundering background checks.
- `isSanctionsScreened : Bool`: Subject is actively cleared against global sanctions databases (OFAC, EU, UN).

### 3.3 Post-Deposit Tainted Wallet Protections
If a depositor's wallet becomes sanctioned or tainted *after* depositing into the vault:
1. **Sanctions Blacklist & Account Freeze**: Compliance officers can update `sanctionedParties` or `frozenParties` on `VaultState`, blocking all future deposits or redemptions by the tainted account.
2. **Forced Escrow Liquidation (`SanctionedEscrowHolding`)**: Compliance officers can exercise `VaultState_ForceRedeemSanctioned` under court/regulatory order, forcibly redeeming the sanctioned party's shares and transferring the underlying assets into a legally isolated `SanctionedEscrowHolding` UTXO in legal custody, keeping the active vault asset pool clean and compliant.

---

## 4. Module Specifications & Function Reference

---

### Module `CantonVault.VaultState`

The `VaultState` template represents the active master state machine of a Canton Vault instance.

#### Template Fields
- `config : VaultConfig`: Vault configuration (provider, underlying instrument ID, share instrument ID, decimals offset, regulator observers).
- `totalAssets : Decimal`: Total assets under management (idle pool + deployed strategy assets).
- `totalShares : Decimal`: Total outstanding vault share token supply.
- `isPaused : Bool`: Emergency pause flag.
- `isPermissioned : Bool`: Enforces explicit institutional allowlisting.
- `allowlist : [Party]`: Approved institutional depositor parties.
- `sanctionedParties : [Party]`: Blacklisted sanctioned parties.
- `frozenParties : [Party]`: Tainted/frozen parties blocked from transacting.
- `observers : [Party]`: List of privacy-approved observers (e.g., depositors).
- `publicObservers : [Party]`: Network-wide public observers for NAV queries.
- `vaultAssetHoldingCid : Optional (ContractId SimpleHolding)`: Contract ID of the physical idle asset pool UTXO.
- `strategyAssets : Decimal`: Total assets currently deployed across external yield strategy adapters.

---

#### Functions & Choices

##### 1. `VaultState_Deposit`
- **Controllers**: `[depositor, config.vaultId.provider]`
- **Arguments**:
  - `depositor : Party`: The depositing party.
  - `inputAssetHoldingCid : ContractId SimpleHolding`: Depositor's input asset holding UTXO.
  - `depositAmount : Decimal`: Amount of underlying assets to deposit.
  - `complianceAttestationCid : Optional (ContractId ComplianceAttestation)`: Attestation CID for permissioned deposits.
- **Preconditions / Assertions**:
  - Vault is not paused (`not isPaused`).
  - Depositor is not on `sanctionedParties` or `frozenParties`.
  - If `isPermissioned`, depositor must be on `allowlist` and present a valid, active `ComplianceAttestation` (`isKycApproved`, `isAmlCleared`, `isSanctionsScreened`).
  - `depositAmount > 0.0`.
  - Holding owner, instrument, and balance match parameters.
- **Expected Behavior**:
  1. Consumes `inputAssetHoldingCid` and creates change holding for depositor if partial spend.
  2. Merges `depositAmount` into `vaultAssetHoldingCid` physical idle asset pool UTXO.
  3. Mints `SimpleHolding` of share tokens for depositor based on `convertToShares`.
  4. Re-creates `VaultState` with updated `totalAssets` and `totalShares`.

---

##### 2. `VaultState_Redeem`
- **Controllers**: `[redeemer, config.vaultId.provider]`
- **Arguments**:
  - `redeemer : Party`: The redeeming party.
  - `inputShareHoldingCid : ContractId SimpleHolding`: Redeemer's input share holding UTXO.
  - `sharesToRedeem : Decimal`: Amount of vault shares to redeem.
- **Preconditions / Assertions**:
  - Vault is not paused (`not isPaused`).
  - Redeemer is not on `sanctionedParties` or `frozenParties`.
  - `sharesToRedeem > 0.0`.
  - Share holding owner, instrument, and balance match parameters.
- **Expected Behavior**:
  1. Consumes `inputShareHoldingCid` and creates change share holding if partial redemption.
  2. Calculates `assetsToReturn` via `convertToAssets`.
  3. Withdraws `assetsToReturn` from physical idle asset pool UTXO.
  4. Mints underlying asset `SimpleHolding` for redeemer.
  5. Re-creates `VaultState` with updated `totalAssets` and `totalShares`.

---

##### 3. `VaultState_ForceRedeemSanctioned`
- **Controllers**: `config.vaultId.provider, sanctionedParty`
- **Arguments**:
  - `sanctionedParty : Party`: The target sanctioned party.
  - `inputShareHoldingCid : ContractId SimpleHolding`: Sanctioned party's share holding UTXO.
  - `sharesToRedeem : Decimal`: Amount of shares to forcibly liquidate.
  - `reason : Text`: Regulatory/court order justification string.
- **Preconditions / Assertions**:
  - Target party must be on `sanctionedParties` or `frozenParties`.
  - `sharesToRedeem > 0.0`.
  - Share holding owner matches `sanctionedParty`.
- **Expected Behavior**:
  1. Archives target sanctioned share holding UTXO.
  2. Withdraws equivalent underlying assets from vault idle pool.
  3. Creates `SanctionedEscrowHolding` UTXO in legal custody of compliance officer.
  4. Re-creates `VaultState` with updated `totalAssets` and `totalShares`.

---

##### 4. `VaultState_HarvestYield`
- **Controller**: `config.vaultId.provider`
- **Arguments**:
  - `yieldAmount : Decimal`: Yield earned in underlying asset units.
  - `returnedYieldHoldingCid : Optional (ContractId SimpleHolding)`: Yield physical holding UTXO.
- **Expected Behavior**:
  1. Merges physical yield token UTXO into vault asset pool.
  2. Increases `totalAssets += yieldAmount` without minting new shares.
  3. Boosts share exchange rate (`sharePrice`) for all vault share holders.

---

##### 5. `VaultState_ProposeStrategyAllocation`
- **Controller**: `config.vaultId.provider`
- **Arguments**:
  - `strategyParty : Party`: Yield Strategy Adapter party.
  - `strategyId : Text`: External strategy pool identifier.
  - `amountToAllocate : Decimal`: Asset amount to deploy.
- **Expected Behavior**:
  1. Withdraws `amountToAllocate` from idle physical asset pool UTXO.
  2. Creates 2-Step `StrategyAllocationRequest` tracking contract.
  3. Increases `strategyAssets += amountToAllocate` while keeping `totalAssets` constant.

---

##### 6. `VaultState_DeallocateFromStrategy`
- **Controller**: `config.vaultId.provider`
- **Arguments**:
  - `principalAmount : Decimal`: Principal amount returned from strategy.
  - `returnedPrincipalHoldingCid : ContractId SimpleHolding`: Returned physical token holding.
- **Expected Behavior**:
  1. Restocks returned principal holding into vault idle physical asset pool UTXO.
  2. Decreases `strategyAssets -= principalAmount` while keeping `totalAssets` constant.

---

## 5. Verification & Test Suite (`CantonVault.Test.VaultSpec`)

The test suite consists of 8 comprehensive Daml Script tests:

| Test Name | Description | Status |
| :--- | :--- | :--- |
| `testVaultLifecycle` | Verifies end-to-end deposit, yield harvest, exchange rate growth, and redemption via 2-step workflows. | **PASSED** |
| `testInstitutionalComplianceAndSanctions` | Verifies regulator observers, KYC attestations, allowlists, sanctions blacklisting, and forced escrow liquidations. | **PASSED** |
| `testStrategyYieldCycle` | Verifies 2-step strategy asset allocation, external yield generation, principal de-allocation, and NAV boost. | **PASSED** |
| `testHarvestYieldUTXOMismatchFails` | Verifies security assertion rejecting declared yield amounts mismatching physical UTXOs. | **PASSED** |
| `testStrategyDeallocationAndLiquidityRebalance` | Verifies restocking of vault idle liquidity pool from strategy de-allocations. | **PASSED** |
| `testPublicNAVQuery` | Verifies network-wide public NAV query access for unprivileged network observers. | **PASSED** |
| `testPartialOperationsAndNAV` | Verifies partial UTXO spends (change holdings), partial share redemptions, and exact exchange rate calculations. | **PASSED** |
| `testInflationProtection` | Verifies immunity against micro-deposit inflation dilution attacks via virtual share offsets ($10^3$). | **PASSED** |
