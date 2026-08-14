# Canton EVM ERC-4626 Tokenized Vault

An enterprise-grade, privacy-preserving **EVM ERC-4626 Tokenized Vault** standard built natively for the **Canton Web3 Network** using Daml smart contracts and the **CIP-0056 / CIP-0112 / CIP-0086** token interface standards.

Incorporates architecture inspired by Ethereum **Morpho Vaults (MetaMorpho)**, featuring virtual share offset inflation protection, multi-market strategy allocation queues, idle physical asset pooling, and public network NAV disclosure.

---

## Key Architecture & Features

1. **CIP-0056 Compliant Token Interface**:
   - Vault share tokens (`vUSDC`) and underlying assets (`USDC`) conform to `Splice.Api.Token.HoldingV1` token interfaces.
2. **Inflation Protection ($10^3$ Virtual Share Offset)**:
   - Uses OpenZeppelin ERC-4626 virtual offset math ($10^3$ virtual shares / $1$ virtual asset), rendering first-depositor share dilution and inflation attacks mathematically impossible.
3. **Canton UTXO Immunity**:
   - Dual-signatory token model (`admin, owner`) prevents un-requested token push transfers, offering protocol-level protection against donation attacks.
4. **Morpho-Style Total Asset Accounting**:
   $$\text{TotalAssets} = \text{Idle Physical Pool Assets} + \sum \text{Strategy Deployed Assets}$$
5. **2-Step Strategy Allocation & De-allocation**:
   - Asynchronous 2-step allocation proposal/acceptance flow between `VaultProvider` and `StrategyAdapter` for strategy deployments and liquidity restocking.
6. **Public NAV Network Observer (`PublicVaultNAV`)**:
   - Publishes aggregate real-time metrics (`totalAssets`, `totalShares`, `sharePrice`, `idleAssets`, `strategyAssets`) to public observers without exposing private user UTXOs or deposit histories.

---

## Directory Structure

```text
daml/
├── CIP0056/
│   └── SimpleHolding.daml      # CIP-0056 Token Holding implementation
└── ERC4626/
    ├── Types.daml              # VaultConfig, VaultStats, PublicNAVData, Virtual Offset Math
    ├── PublicNAV.daml          # Network-wide Public NAV disclosure template
    ├── Strategy.daml           # 2-step Strategy Allocation & De-allocation workflows
    ├── VaultState.daml         # Master Vault state machine, deposits, harvests, redemptions
    ├── DepositWorkflow.daml    # 2-step Deposit Request workflow
    ├── RedeemWorkflow.daml     # 2-step Redeem Request workflow
    └── Test/
        └── VaultSpec.daml      # Comprehensive Daml Script test suite (7 tests)
```

---

## Prerequisites & Building

Requires the **Canton DPM CLI** (`dpm`):

```bash
# Build package DAR
dpm build

# Run unit tests
dpm test
```

---

## License

Apache-2.0
