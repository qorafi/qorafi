# QoraFi Protocol
<p align="center">
<img width="2000" height="600" alt="curve" src="https://github.com/user-attachments/assets/a5f84f63-a804-4aca-a1dd-c44c6897ceeb" />
</p>

**A decentralized, secure, and transparent ecosystem for liquidity bootstrapping, staking, and stablecoin minting for the QoraFi token.**

---
<p align="center">
<img width="512" height="702" alt="download (1)" src="https://github.com/user-attachments/assets/e4c152c8-a633-4511-a30f-3cec85e596f1" />

</p>
## Abstract

The QoraFi Protocol introduces a decentralized, secure, and transparent ecosystem for the QoraFi token, encompassing liquidity bootstrapping, staking, and stablecoin minting. The system is architected in a modular fashion, separating core business logic from security and oracle services to enhance robustness and auditability. The protocol consists of several primary smart contracts that work in concert to create a resilient and feature-rich DeFi ecosystem.

## System Architecture

The protocol is designed with a clear separation of concerns, dividing responsibilities among specialized, interoperable smart contracts. This modular design enhances security, simplifies auditing, and allows for safer future upgrades.

* **`SecurityManager.sol`**: A foundational base contract that provides robust, reusable security features. It handles access control, pausing, reentrancy protection, anti-MEV (Miner Extractable Value) mechanisms, and a volume-based circuit breaker.
* **`MarketOracle.sol`**: A dedicated and hardened oracle contract responsible for calculating a secure and manipulation-resistant price for the QoraFi token. It uses a multi-observation Time-Weighted Average Price (TWAP) from a Uniswap V2-style liquidity pool.
* **`BondingCurve.sol`**: The primary user-facing contract for initial liquidity formation. It facilitates deposits of stablecoins (USDT), BNB, or other "zap-in" tokens, swaps a portion for QoraFi, and pairs the remainder to create and distribute LP (Liquidity Provider) tokens.
* **`ProofOfLiquidity.sol`**: A secure vault contract whose sole responsibility is to hold users' staked QoraFi/USDT LP tokens. It handles the core `stake` and `unstake` logic.
* **`RewardEngine.sol`**: A dedicated reward management contract for the Proof of Liquidity system. It contains all the complex logic for calculating and distributing staking rewards, including compounding interest and a quarterly reward decay.

## Core Features

* **Modular & Upgradable Design**: Core logic is separated from security, oracle, and reward modules, allowing for safer and more flexible governance and upgrades.
* **Hardened Oracle Security**: Utilizes a multi-observation TWAP mechanism to provide a manipulation-resistant price feed, complete with liquidity checks and price deviation circuit breakers.
* **Anti-MEV Protection**: Implements block interval limits and per-block deposit caps to mitigate front-running and sandwich attacks during liquidity formation.
* **Compounding Staking Rewards**: The Proof of Liquidity system allows stakers to earn rewards on their principal stake plus their previously accrued, unclaimed rewards.
* **Sustainable Tokenomics**: A quarterly reward decay schedule is built into the `RewardEngine` to ensure a predictable and sustainable inflation model.
* **Decentralized Governance**: The entire protocol is designed to be owned and managed by a DAO via a `TimelockController`, with granular, role-based access control for all critical functions.
* **Emergency Safeguards**: Includes a system-wide circuit breaker, pausable contracts, and a timelocked emergency function system for the DAO to act in unforeseen circumstances.

## Contract Details

### `SecurityManager.sol`

This contract is inherited by `BondingCurve.sol` and provides a suite of security features:

* **Access Control**: Manages `GOVERNANCE_ROLE`, `PAUSER_ROLE`, and `EMERGENCY_ROLE`.
* **Anti-MEV**: Enforces deposit frequency and volume limits.
    * **Block Interval Limit**: `block.number - lastDepositBlock[user] >= minDepositInterval`
    * **Per-Block Volume Limit**: `blockDepositTotal[block.number] + depositAmount <= maxDepositPerBlock`
* **Circuit Breaker**: Halts deposits if volume exceeds a predefined threshold in a short time window.
* **Emergency Timelock**: A two-step process for executing critical actions after a delay.
    * `queueEmergencyTransaction(...)`: Proposes an action.
    * `executeEmergencyTransaction(...)`: Executes the action after a timelock period.
    * **Execution Time Formula**: `$executeAfter = block.timestamp + EMERGENCY\_TIMELOCK\_DELAY$`

### `MarketOracle.sol`

The dedicated price feed for the protocol. It is designed to be highly resistant to price manipulation.

* **Core Features**:
    * **Multi-Observation TWAP**: Uses a weighted average of multiple price observations to resist manipulation.
    * **Price Validation**: Includes circuit breakers that validate any new price against the last known price and market cap.
    * **Liveness & Liquidity Checks**: Ensures the underlying liquidity pool is healthy before providing a price.
* **Key Functions & Formulas**:
    * **`updateMarketCap()`**: The core keeper function to update the oracle's state.
    * **`_getEnhancedTWAPPrice()`**: Calculates the time-weighted average price.
        $$
        Price_{TWAP} = \frac{\sum_{i=1}^{n} (Price_i \times TimeElapsed_i)}{\sum_{i=1}^{n} TimeElapsed_i}
        $$
    * **`_calculateTwapPrice(...)`**: Converts Uniswap's cumulative price to a readable format.
        * **Formula (if QoraFi is `token1`)**: $Price = \frac{PriceCumulativeDiff}{TimeElapsed} \times \frac{10^{Decimals_{USDT}}}{2^{112}}$
        * **Formula (if QoraFi is `token0`)**: $Price = \frac{2^{112} \times 10^{Decimals_{USDT}}}{(\frac{PriceCumulativeDiff}{TimeElapsed})}$
    * **`_validatePriceChange(...)`**: Circuit breaker for price volatility.
        * **Formula**: $\frac{|newPrice - oldPrice|}{oldPrice} \times 10000 \le maxPriceChangeBPS$
    * **`_validateMarketCapGrowth(...)`**: Circuit breaker for market cap growth.
        * **Formula**: $\frac{newCap - oldCap}{oldCap} \times 10000 \le maxMarketCapGrowthBPS$

### `BondingCurve.sol`

The main entry point for users to acquire QoraFi and provide liquidity.

* **`deposit(...)`**: The primary function for depositing USDT.
* **`depositWithBNB(...)` / `depositWithToken(...)`**: "Zap-in" functions that allow users to deposit with BNB or other whitelisted tokens.
* **`_startBondingProcess(...)`**: The internal orchestrator that checks oracle limits, splits the deposit, swaps for QoraFi, and adds liquidity.
* **`_swapAndSplit(...)`**: Splits the user's deposit and calculates a secure minimum output for the swap.
    * **Slippage Protection Formula**: $minOutput = expectedOutput \times \frac{10000 - maxSlippageBPS}{10000}$

### `ProofOfLiquidity.sol` (The Vault)

A simple and secure vault for holding staked LP tokens.

* **`stake(uint256 _amount)`**: Deposits LP tokens and notifies the `RewardEngine`.
* **`unstake(uint256 _amount)`**: Withdraws LP tokens and notifies the `RewardEngine`.
* **`emergencyUnstake()`**: Allows for immediate withdrawal of principal LP tokens, with a reward penalty applied in the `RewardEngine`.

### `RewardEngine.sol`

Manages all staking reward calculations and distribution.

* **`handleStakeChange(...)`**: Called by the `ProofOfLiquidity` vault to update a user's collateral value.
* **`claimMyRewards(...)`**: The user-facing function to claim accrued rewards, which triggers the minting of new QoraFi tokens.
* **`getCurrentDailyRewardPercentageBPS()`**: A view function that calculates the current reward rate based on the quarterly decay schedule.

## Getting Started

To set up this project locally, you will need a development environment like Hardhat or Foundry.

1.  **Clone the repository:**
    ```bash
    git clone <your-repo-url>
    cd <your-repo-directory>
    ```
2.  **Install dependencies:**
    ```bash
    npm install
    ```
3.  **Compile the contracts:**
    ```bash
    npx hardhat compile
    ```

## Usage Example

A typical user interaction with the protocol would be:

1.  **Approve USDT**: The user first approves the `BondingCurve` contract to spend their USDT.
2.  **Deposit**: The user calls the `deposit` function on the `BondingCurve` contract with the amount of USDT they wish to contribute.
3.  **Receive LP Tokens**: The `BondingCurve` contract automatically swaps half of the USDT for QoraFi, pairs it with the remaining USDT, adds the liquidity to PancakeSwap, and sends the resulting LP tokens to the user.
4.  **Stake LP Tokens**: The user then calls the `stake` function on the `ProofOfLiquidity` contract to stake their newly acquired LP tokens and start earning rewards.

## Security

The QoraFi Protocol has been designed with security as the highest priority. Key security features include:

* **Modular Architecture**: Isolates critical components to minimize attack surface.
* **Role-Based Access Control**: Granular permissions for all administrative functions.
* **Timelock Governance**: Ensures all critical changes are subject to a time delay, allowing for community review.
* **Manipulation-Resistant Oracles**: Uses a multi-observation TWAP with circuit breakers.
* **MEV Protection**: Mitigates front-running and sandwich attacks.
* **Reentrancy Guards**: All critical state-changing functions are protected against reentrancy.
* **Emergency Mechanisms**: Includes pausing, circuit breakers, and a timelocked emergency execution system.

## Contributing

Contributions are welcome! Please feel free to open an issue or submit a pull request.

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.
