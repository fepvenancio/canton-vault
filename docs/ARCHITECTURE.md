# Canton Vault Architecture & Technical Specification

This document provides a comprehensive technical manual for **Canton Vault** (`canton-vault`), detailing mathematical foundations, ledger privacy patterns, template definitions, and exhaustive function specifications.

---

## 1. Executive Summary & Design Philosophy

**Canton Vault** is a tokenized asset yield vault built natively for the **Canton Web3 Network** using Daml smart contracts. It integrates with Canton's official token standards (**CIP-0056**, **CIP-0112**, **CIP-0086**) implementing:

1. **Virtual Share Offset Math ($10^3$)**: Prevents inflation dilution attacks.
2. **Dual-Signatory UTXO Security**: Protects against un-requested token donation attacks.
3. **Canton Privacy vs. Public Disclosure**: Preserves user privacy on individual holdings while exposing real-time aggregate Net Asset Value (NAV) metrics to public network observers.
4. **Morpho-Style Balance Sheet Invariant**:
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
- **Expected Behavior**: Truncates output at Daml `Decimal` fixed precision ($10^{-10}$ atomic units), ensuring rounding always favors the vault over the redeemer.

---

## 3. Module Specifications & Function Reference

---

### Module `CantonVault.VaultState`

The `VaultState` template represents the active master state machine of a Canton Vault instance.

#### Template Fields
- `config : VaultConfig`: Vault configuration (provider, underlying instrument ID, share instrument ID, decimals offset).
- `totalAssets : Decimal`: Total assets under management (idle pool + deployed strategy assets).
- `totalShares : Decimal`: Total outstanding vault share token supply.
- `isPaused : Bool`: Emergency pause flag.
- `observers : [Party]`: List of privacy-approved observers (e.g., depositors).
- `publicObservers : [Party]`: Network-wide public observers for NAV queries.
- `vaultAssetHoldingCid : Optional (ContractId SimpleHolding)`: Contract ID of the physical idle asset pool UTXO.
- `strategyAssets : Decimal`: Total assets currently deployed across external yield strategy adapters.

---

#### Functions & Choices

##### 1. `VaultState_Deposit`
- **Controller**: `depositor : Party`
- **Arguments**:
  - `depositor : Party`: The depositing party.
  - `inputAssetHoldingCid : ContractId SimpleHolding`: Depositor's input asset holding UTXO.
  - `depositAmount : Decimal`: Amount of underlying assets to deposit.
- **Preconditions / Assertions**:
  - `isPaused` must be `False`.
  - `depositAmount > 0.0`.
  - `holding.owner == depositor`.
  - `holding.instrumentId == config.underlyingInstrument`.
  - `holding.amount >= depositAmount`.
  - `sharesToMint > 0.0`.
- **Internal Logic & Expected Behavior**:
  1. Consumes `inputAssetHoldingCid`.
  2. If `holding.amount > depositAmount`, creates a change UTXO for `depositor`.
  3. Merges `depositAmount` into `vaultAssetHoldingCid` idle physical asset pool.
  4. Calculates `sharesToMint` via `convertToShares`.
  5. Mints `SimpleHolding` of vault share tokens (`config.shareInstrument`) for `depositor`.
  6. Re-creates `VaultState` with updated `totalAssets += depositAmount` and `totalShares += sharesToMint`.

---

##### 2. `VaultState_Redeem`
- **Controller**: `redeemer : Party`
- **Arguments**:
  - `redeemer : Party`: The redeeming party.
  - `inputShareHoldingCid : ContractId SimpleHolding`: Redeemer's input share holding UTXO.
  - `sharesToRedeem : Decimal`: Amount of shares to redeem.
- **Preconditions / Assertions**:
  - `isPaused` must be `False`.
  - `sharesToRedeem > 0.0`.
  - `shareHolding.owner == redeemer`.
  - `shareHolding.instrumentId == config.shareInstrument`.
  - `shareHolding.amount >= sharesToRedeem`.
  - `assetsToReturn <= totalAssets`.
  - Physical idle pool `poolHolding.amount >= assetsToReturn`.
- **Internal Logic & Expected Behavior**:
  1. Consumes `inputShareHoldingCid`.
  2. If `shareHolding.amount > sharesToRedeem`, creates share change UTXO for `redeemer`.
  3. Calculates `assetsToReturn` via `convertToAssets`.
  4. Withdraws `assetsToReturn` from `vaultAssetHoldingCid` idle pool.
  5. Mints underlying asset `SimpleHolding` for `redeemer`.
  6. Re-creates `VaultState` with updated `totalAssets -= assetsToReturn` and `totalShares -= sharesToRedeem`.

---

##### 3. `VaultState_HarvestYield`
- **Controller**: `config.vaultId.provider : Party`
- **Arguments**:
  - `yieldAmount : Decimal`: Amount of yield generated by strategy.
  - `returnedYieldHoldingCid : Optional (ContractId SimpleHolding)`: Physical yield token holding UTXO.
- **Preconditions / Assertions**:
  - `isPaused` must be `False`.
  - `yieldAmount > 0.0`.
  - `yieldHolding.owner == provider`.
  - `yieldHolding.instrumentId == config.underlyingInstrument`.
  - `yieldHolding.amount == yieldAmount` (strictly enforced to prevent UTXO/accounting desynchronization).
- **Internal Logic & Expected Behavior**:
  1. Merges `returnedYieldHoldingCid` into `vaultAssetHoldingCid` idle pool.
  2. Re-creates `VaultState` with `totalAssets += yieldAmount`.
  3. Increases share price ($\frac{\text{totalAssets}}{\text{totalShares}}$) automatically for all share holders.

---

##### 4. `VaultState_ProposeStrategyAllocation`
- **Controller**: `config.vaultId.provider : Party`
- **Arguments**:
  - `strategyParty : Party`: Strategy adapter party.
  - `strategyId : Text`: Identifier for strategy market.
  - `amountToAllocate : Decimal`: Principal amount to deploy.
- **Preconditions / Assertions**:
  - `isPaused` must be `False`.
  - `amountToAllocate > 0.0`.
  - Idle pool `poolHolding.amount >= amountToAllocate`.
- **Internal Logic & Expected Behavior**:
  1. Withdraws `amountToAllocate` from `vaultAssetHoldingCid` idle pool.
  2. Creates a 2-step `StrategyAllocationRequest` tracking contract observed by `strategyParty`.
  3. Re-creates `VaultState` with `strategyAssets += amountToAllocate`.

---

##### 5. `VaultState_DeallocateFromStrategy`
- **Controller**: `config.vaultId.provider : Party`
- **Arguments**:
  - `principalAmount : Decimal`: Principal amount returned from strategy.
  - `returnedPrincipalHoldingCid : ContractId SimpleHolding`: Returned physical token holding UTXO.
- **Preconditions / Assertions**:
  - `principalAmount > 0.0`.
  - `principalAmount <= strategyAssets`.
  - `principalHolding.owner == provider`.
  - `principalHolding.instrumentId == config.underlyingInstrument`.
  - `principalHolding.amount == principalAmount`.
- **Internal Logic & Expected Behavior**:
  1. Merges `returnedPrincipalHoldingCid` back into `vaultAssetHoldingCid` idle pool.
  2. Re-creates `VaultState` with `strategyAssets -= principalAmount`.
  3. Restores idle physical liquidity for user redemptions.

---

##### 6. `VaultState_GetStats` & `VaultState_GetPublicNAVData`
- **Controller**: `caller : Party`
- **Non-consuming choices**: Return real-time exchange rate statistics (`VaultStats`) and public balance sheet breakdown (`PublicVaultNAVData`).

---

### Module `CantonVault.Strategy`

Manages 2-step strategy fund deployment, yield accrual, and principal de-allocations.

#### 1. `StrategyAllocationRequest_Accept`
- **Controller**: `strategyParty : Party` (with `readAs provider`)
- **What it does**: Strategy accepts allocated funds from `VaultProvider`, archiving `allocatedAssetHoldingCid`, creating strategy-owned `SimpleHolding`, and initializing `StrategyAllocation` UTXO.

#### 2. `Strategy_DeallocatePrincipal`
- **Controller**: `strategyParty : Party`
- **What it does**: Strategy returns principal holding back to `VaultProvider` for idle pool restocking.

#### 3. `Strategy_ReturnAssetsAndYield`
- **Controller**: `strategyParty : Party`
- **What it does**: Strategy returns principal and accrued yield back to `VaultProvider`.

---

### Module `CantonVault.PublicNAV`

Provides transparent public network disclosure.

#### `PublicVaultNAV_GetStats`
- **Controller**: `caller : Party` (any unprivileged observer party)
- **What it does**: Queries `PublicVaultNAVData` (`totalAssets`, `totalShares`, `sharePrice`, `idleAssets`, `strategyAssets`) without exposing individual depositor UTXOs or private trade histories.

---

### Modules `CantonVault.DepositWorkflow` & `CantonVault.RedeemWorkflow`

Provide asynchronous 2-step workflows (`DepositRequest` / `RedeemRequest`) allowing depositors and redeemers to propose vault actions that `VaultProvider` executes on-chain.

---

## 4. Invariants & Security Matrix

1. **Share Price Invariant**:
   $$\text{SharePrice} = \frac{\text{TotalAssets}}{\text{TotalShares}}$$
2. **Balance Sheet Invariant**:
   $$\text{TotalAssets} = \text{IdleAssets} + \text{StrategyAssets}$$
3. **UTXO Equality Invariant**:
   $$\text{vaultAssetHoldingCid.amount} = \text{IdleAssets}$$
