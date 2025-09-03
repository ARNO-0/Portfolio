### H[01] LinearPriceModel: _getPrice double‑counts slope, inflating spot price

### Summary

LinearPriceModel._getPrice double‑counts the slope term when computing nextPrice, causing the spot price (and thus the buyer’s cost) to rise faster than the advertised linear curve.

### Finding Description
```
// buggy line in _getPriceint256 
nextPrice =    priceSlope * int256(sale.sold + quantity) / 1e18 + prevPrice;
```

prevPrice already equals
initialPrice + priceSlope × sale.sold / 1e18.
Adding the same component again evaluates to:

nextPrice_bug =  initialPrice+ 2 × priceSlope × sale.sold   / 1e18   // double‑count+     priceSlope × quantity    / 1e18

The correct linear expression should be:

nextPrice_ok =  initialPrice+ priceSlope × (sale.sold + quantity) / 1e18

The error grows linearly with sale.sold; the deeper into the sale, the larger the over‑charge.

### Impact Explanation


Over‑charging users. Every purchase after the first token is priced above the public curve. At late stages the premium exceeds 70 % per token.


### Likelihood Explanation

The bug triggers every time getPriceAt/getPriceNow is called after at least one token is sold.

### Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/**
 * PoC for LinearPriceModel bug.
 * Parameters: 1 000‑token sale, 800 sold, buyer wants 100 more.
 */
contract PriceModelPoC {
    uint256 constant INITIAL = 1 ether;  // 1e18 wei
    uint256 constant MAX     = 5 ether;  // 5e18 wei
    uint256 constant Q_TOTAL = 1_000e18; // 1 000 tokens (18 dec)
    uint256 constant SOLD    = 800e18;   // 80 % progress
    uint256 constant BUY_QTY = 100e18;   // purchase size

    // slope scaled by 1e18 (matches production code)
    uint256 constant SLOPE = (MAX - INITIAL) * 1e18 / Q_TOTAL; // 4e15 wei

    /// Correct linear formula
    function nextPriceCorrect() public pure returns (uint256) {
        return INITIAL + SLOPE * (SOLD + BUY_QTY) / 1e18; // 4.6 ETH
    }

    /// Buggy production formula
    function nextPriceBuggy() public pure returns (uint256) {
        uint256 prevPrice = INITIAL + SLOPE * SOLD / 1e18; // 4.2 ETH
        return prevPrice + SLOPE * (SOLD + BUY_QTY) / 1e18; // 7.8 ETH
    }

    /// Difference in wei (3.2 ETH)
    function overcharge() external pure returns (uint256) {
        return nextPriceBuggy() - nextPriceCorrect();
    }
}

```

```
forge test
[PASS] nextPriceCorrect == 4.6 ether
[PASS] nextPriceBuggy   == 7.8 ether
[PASS] overcharge       == 3.2 ether

```

### Recommendation

Replace the buggy line with the correct linear expression:

```
// current (buggy)
int256 nextPrice = priceSlope * int256(sale.sold + quantity) / 1e18 + prevPrice;

// fixed
int256 nextPrice = int256(initialPrice)
                 + priceSlope * int256(sale.sold + quantity) / 1e18;

```
### H[02] Unclaimed Rewards Become Permanently Lost After NFT Token Withdrawal

### Summary

The claimRewards function in the ERC20Gauge contract contains a critical bug that leads to permanent loss of user rewards when they withdraw their staked tokens. The root cause is that the withdraw function burns the NFT before rewards can be claimed, while the claimRewards function requires the NFT to exist to identify the rightful recipient.

### Finding Description

```solidity
function claimRewards(uint256 nftId) public nonReentrant {
    // BUG: if nft is burned in withdraw() function then user won't be able to claim rewards
    updateReward(nftId);
    address account = ownerOf(nftId);
    uint256 reward = _rewards[nftId];
    if (reward > 0) {
        _rewards[nftId] = 0;
        // Transfer the reward from the funding account to the account that receives the rewards
        _rewardToken.safeTransferFrom(_fundingAccount, account, reward);
        emit RewardsPaid(nftId, account, reward);
    }
}
```

In this function:

- The code calls `ownerOf(nftId)` to determine who should receive the rewards
- When an NFT has been burned through the withdraw function, `ownerOf(nftId)` will revert with "ERC721: owner query for nonexistent token"
- As a result, any rewards that were accrued but not claimed before withdrawal become permanently locked in the contract

### Impact Explanation

- **Direct Financial Loss**: Users who withdraw their tokens without first claiming rewards will permanently lose all accrued rewards
- **Poor User Experience**: Users might reasonably expect that they can claim rewards after unstaking, but will find their rewards inaccessible
- **Protocol Token Emissions**: Tokens allocated as rewards but never distributed create a discrepancy between intended and actual token emissions

### Likelihood Explanation

This bug triggers whenever a user withdraws their staked tokens without first claiming their rewards. Given that withdrawal and reward claiming are separate functions, users may naturally assume they can claim rewards after unstaking, making this scenario highly likely.

### Proof of Concept

**Scenario:**

1. User Alice stakes 1000 tokens through `deposit()` and receives NFT #1
2. After some time, Alice has earned 50 reward tokens
3. Alice calls `withdraw(1, alice)` to retrieve her original 1000 tokens
   - The withdraw function correctly calls `updateReward(1)` to calculate Alice's rewards
   - Rewards are stored in `_rewards[1]` mapping
   - The NFT #1 is burned: `_burn(1)`
4. Alice later attempts to claim her rewards with `claimRewards(1)`
   - This call reverts at `ownerOf(1)` because the NFT no longer exists
5. Alice's 50 reward tokens are now permanently locked in the contract

### Recommendation

There are two potential solutions:

**Option 1: Automatically claim rewards on withdrawal**


### H[03] Premature State Clearing in FeeDistributor's claim() Function

### Summary

In the claim() function of the FeeDistributor contract, earned amounts are reset to zero before those values are used to update the contract's internal balance tracking. This creates a critical accounting error where the _totalBalances mapping never gets reduced after users claim their fees.



```solidity
// Add earned amounts to claimed amounts
claimedFor.amountToken0 += earnedFor.amountToken0;
claimedFor.amountToken1 += earnedFor.amountToken1;
// Reset earned amounts
earnedFor.amountToken0 = 0; // @issue : state got cleared early
earnedFor.amountToken1 = 0; // @issue : state got cleared early
// These operations now subtract ZERO instead of the claimed amounts
_totalBalances[_tokenPair(factoryId).token0] -= earnedFor.amountToken0; 
_totalBalances[_tokenPair(factoryId).token1] -= earnedFor.amountToken1;
```

### Impact 

- Incorrect Balance Tracking: _totalBalances will continuously overstate the amount of fees owed, since claims never reduce it.
- Minting and InsufficientBalance Checks Break: Future calls to mint use _totalBalances to verify that the contract has enough funds (_totalBalances + amount <= balanceOf(this)). With stale, inflated totals, new mints will incorrectly fail the "InsufficientBalance" check, even if the contract holds sufficient tokens.
- Denial of Service: Over time, legitimate fee claims and mints may be blocked, effectively freezing fee distribution.

### Likelihood 

This bug triggers on every claim operation, making it a high-frequency issue.

### Proof of Concept

1. User Alice has earned 100 tokens of fee rewards
2. Alice calls claim() to withdraw her 100 tokens
3. The tokens are correctly transferred to Alice
4. claimedFor.amountToken0 is correctly updated to 100
5. earnedFor.amountToken0 is reset to 0
6. _totalBalances[token0] is reduced by 0 (not 100)
7. Result: Contract doesn't track that 100 tokens have been claimed and paid out

### Recommendation

Move the state clearing operations after the balance updates:

```solidity
// Add earned amounts to claimed amounts
claimedFor.amountToken0 += earnedFor.amountToken0;
claimedFor.amountToken1 += earnedFor.amountToken1;
// Update total balances BEFORE clearing earned amounts
_totalBalances[_tokenPair(factoryId).token0] -= earnedFor.amountToken0; 
_totalBalances[_tokenPair(factoryId).token1] -= earnedFor.amountToken1; 
// Reset earned amounts AFTER balances are updated
earnedFor.amountToken0 = 0;
earnedFor.amountToken1 = 0;
```



### M[01] Missing Signature Validation for CustomData in TokenFactory

### Description

The TokenFactory contract's create() function contains a critical signature validation flaw where the customData field in the CreateParams structure is not included in the signature verification process. This parameter contains the maximum price value (maxPrice) used by the LinearPriceModel to calculate the price curve for token sales.



When a project owner initiates a token launch with a signed transaction, an attacker can observe the pending transaction, copy all parameters, modify only the customData field, and front-run the original transaction. Since customData isn't validated in the signature check, this modified transaction will be processed as legitimate.

In the LinearPriceModel contract, the customData is decoded to extract the maximum price:

```solidity
// Decode the maximum price from the custom data
uint256 maxPrice = abi.decode(sale.customData, (Price)).toUint160();
// Calculate the price slope based on this maxPrice
int256 priceSlope = maxPrice > initialPrice
    ? int256((maxPrice - initialPrice) * 1e18 / (sale.quantity))
    : -int256((initialPrice - maxPrice) * 1e18 / (sale.quantity));
```

### Impact 

This vulnerability enables several attack vectors:

- Price Manipulation: Attackers can set arbitrary maximum prices that drastically alter the token price curve, potentially:
    - Creating an extremely steep price increase to extract more value from buyers
    - Setting a negative slope (where price decreases with volume) to discourage early participation
    - Setting a flat price curve to eliminate price dynamics that may benefit the project

- Project Economics Disruption: The intended token sale mechanics, carefully designed by project owners, can be completely undermined.

- Investor Harm: Participants in the token sale may face unexpected price dynamics, potentially paying more than they should or receiving fewer tokens than expected.

- Brand Damage: Projects may face reputational damage if their token launches operate under manipulated price mechanics that they didn't authorize.

### Likelihood 

This attack is highly feasible as it requires only monitoring the mempool for token creation transactions and front-running them with modified parameters.

### Proof of Concept

1. A legitimate project owner creates and signs a transaction to launch a token with:
    - initialPrice: 0.001 ETH
    - customData: Containing an encoded maximum price of 0.002 ETH (reasonable 2x increase)

2. An attacker observes this pending transaction in the mempool and creates a new transaction with:
    - Identical parameters to the original transaction
    - Modified customData: Containing an encoded maximum price of 0.1 ETH (100x increase)

3. The attacker submits this transaction with higher gas, front-running the legitimate transaction

4. The contract processes the attacker's transaction:
    - The signature validation passes because customData isn't included in the verification
    - The token is created with the manipulated price parameters
    - The original transaction fails due to GardenId already being used

5. The token sale proceeds with an extremely steep price curve that wasn't intended by the project owner.

### Recommendation

Include the customData field in the signature validation process