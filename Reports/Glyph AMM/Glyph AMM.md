![367495409-c589c5d6-f32d-444f-9eb2-bb13a4c3836f](https://github.com/user-attachments/assets/2cbdebed-0c11-4655-a2e6-d77ca761d52a)


## Title
Incorrect Identification of WETH as a Fairc20 Token Leading to Pair Creation Reversion

## Description
The `SwapFactory` contract is designed to facilitate the creation of token pairs for a decentralized exchange. A significant issue arises when users attempt to create a pair involving Wrapped Ether (WETH) and any other token, due to WETH’s behavior of accepting arbitrary calls and returning true. The contract includes a mechanism intended to restrict pair creation to only whitelisted addresses for certain tokens identified as Fairc20. However, because WETH on the Ethereum mainnet will return true for any call, including the `mintComplete()` check used to identify Fairc20 tokens, attempts to create pairs with WETH and another token (e.g., USDT) will inadvertently be rejected if `isFairc20PoolWhitelistEnabled` is enabled. By default, this setting is true upon deployment, leading to an unexpected behavior where valid pair creation transactions are reverted despite the creator having no intention of establishing a Fairc20 pair.

## Impact
This bug directly affects the user experience and functionality of the `SwapFactory` contract, restricting liquidity provision opportunities and potentially undermining the platform’s usability. Users intending to create pairs with WETH and other legitimate tokens will face transaction rejections, thereby limiting the diversity and richness of the trading ecosystem. Additionally, this could erode trust in the platform, as users might not understand the technical reason behind these rejections and perceive the platform as unreliable or overly restrictive.

## Proof of Concept
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/SwapFactory.sol";
import "../src/ERC20Mock.sol";

contract SwapFactoryTest is Test {
    SwapFactory swapFactory;
    ERC20Mock tokenA;
    address WETH_TOKEN = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    uint256 mainnetFork;

    function setUp() public {
        mainnetFork = vm.createFork("", 19452615);
        vm.selectFork(mainnetFork);
        // Deploy Mock ERC20 tokens
        tokenA = new ERC20Mock("", "", 10000); // Assuming the constructor takes initial supply

        // Deploy the SwapFactory and set up the initial state as needed
        address feeToSetter = address(this);
        swapFactory = new SwapFactory(feeToSetter);

        // Set up other state variables if necessary
        swapFactory.setIsCreatePoolWhitelistEnabled(false);
        (bool token0IsFairc20, ) = WETH_TOKEN.call(
            abi.encodeWithSelector(bytes4(keccak256("mintComplete()")))
        );
        console.log(token0IsFairc20);
    }

    function testCreatePairExpectRevert() public {
        vm.expectRevert("GlyphDEX Swap: Not Whitelist");
        // Attempt to create a pair that should fail under test conditions
        swapFactory.createPair(address(tokenA), WETH_TOKEN);
    }
}
```

## Tools Used
VSCode

## Recommendation
Adjust the Fairc20 detection method.
