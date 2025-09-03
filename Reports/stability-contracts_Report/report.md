### M[01] Receiver-Side Cool-Down Reset Lets Attacker DoS Any Account’s Withdrawals via Dust depositAssets Calls

#### Description

`VaultBase` implements a cool-down meant to stop a user from withdrawing early. The mechanism is:

```solidity
// 1. every withdrawal
if (withdrawRequests[owner] + 5 >= block.number) revert WaitAFewBlocks();
withdrawRequests[owner] = block.number;

// 2. every deposit
withdrawRequests[receiver] = block.number;   // <─ problem line
```

Because any address can be set as `receiver` in `depositAssets`, an attacker can continuously deposit 1 wei of the vault’s cheapest asset to a victim’s address. Each dust deposit resets `withdrawRequests[victim]` to the current block, so the test `withdrawRequests[victim] + 5 >= block.number` is perpetually true and every `withdrawAssets` call made by that victim reverts with `WaitAFewBlocks()`.

#### Attack Outline

1.  Attacker holds a tiny amount of any token accepted by the vault.
2.  Each block (or on demand) attacker calls:
    ```solidity
    vault.depositAssets([tokenX], [1], 0, victim);  // 1 wei, minSharesOut = 0
    ```
3.  Victim’s withdrawal attempts revert indefinitely. The griefing stops only if governance disables the defense (`setLastBlockDefenseDisabled`), which also removes the MEV protection for everyone.

**Cost to attacker:** Negligible
**Impact:** Denial of Service (DoS)

**NOTE:** THE SAME ISSUE EXISTS IN THE `MetaVault` CONTRACT.

#### Recommendation

Do not update the cool-down for the deposit `receiver`.

-----

### M[02] Safety Fuse Can Be Bypassed to Manipulate Share Price

#### Description

`VaultBase.depositAssets()` is meant to stop deposits when the strategy has no value backing the outstanding shares, because minting against a zero (or near-zero) denominator would wildly dilute everyone.

```solidity
if (v._totalSupply != 0 && v.totalValue == 0) {
    revert FuseTrigger();                      // “safety fuse”
}
```

Where:

  * `v._totalSupply` = current vault share supply
  * `v.totalValue` = `strategy.total()` – the strategy’s Balance of liquidity token

#### How the Fuse is Bypassed

1.  **Initial state** – The vault already has shares in circulation (`_totalSupply > 0`), but the strategy’s underlying position has been closed or rugged, leading to `strategy.total() == 0`. Deposits are rightly blocked.

2.  **Attacker action** – The attacker holds a small amount of the strategy’s liquidity token (from before the fuse was triggered) and transfers it directly to the `ERC4626StrategyBase` contract.

      * The strategy’s bookkeeping doesn’t mint any strategy-internal shares, but `total()` simply sums the raw token balance, so it now returns a non-zero TVL (e.g., 1 wei).

    <!-- end list -->

    ```solidity
    /// @inheritdoc IStrategy
    // returns how much amount of shares this contract is holding of underlying vault
    function total() public view override returns (uint) {
        StrategyBaseStorage storage __$__ = _getStrategyBaseStorage();
        // @note _$__._underlying is a token address and using address this balance
        return StrategyLib.balance(__$__._underlying);
    }
    ```

3.  **Second deposit** – The attacker calls `depositAssets()` again.

      * `v._totalSupply` is still the full share count.
      * `v.totalValue` is now tiny but non-zero.
      * The fuse condition now passes, and the function proceeds to mint: `$mintAmount = (value_ * totalSupply_) / totalValue_$`.
      * Because `totalValue_` is tiny, `mintAmount` becomes enormous relative to `value_`. The attacker obtains the majority of vault shares for a trivial deposit, diluting the vault equity from existing holders.

#### Impact

  * **Share-price manipulation**: The attacker drives the share price near zero, then mints huge amounts.
  * **Value dilution / theft**: Existing users’ shares represent a fraction of the vault’s assets after the attack.
  * **Low cost to exploit**: Needs only a minimal amount of the underlying token and one extra transaction.

#### Recommendation

This issue requires a major refactoring.