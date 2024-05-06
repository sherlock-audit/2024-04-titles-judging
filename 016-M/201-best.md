Clever Lemon Starling

medium

# [M-1] Precision loss due to division before multiplication in `FeeManager._buildSharesAndTargets` (Incorrect Order of Operations)

## Summary

The `_buildSharesAndTargets` function in the `FeeManager.sol` contract performs a division operation before a multiplication operation, which can lead to precision loss. Solidity's integer division truncates the result, so performing division before multiplication can cause rounding errors and result in incorrect share calculations.

## Vulnerability Detail

Add the following code to `FeeManager.t.sol`:

```javascript 
// add this function to FeeManagerTest
 function test_buildSharesAndTargets_divideBeforeMultiply() public {
        Target memory creator = Target({target: address(0xc0ffee), chainId: block.chainid});
        Target[] memory attributions = new Target[](1);
        attributions[0] = Target({target: address(0xdead), chainId: block.chainid});

        uint32 revshareBps = 5000; // 50%

        (address[] memory targets, uint256[] memory shares) =
            exposedFeeManager.exposedBuildSharesAndTargets(creator, attributions, revshareBps);

        // Expected values if multiplication is performed before division
        uint256 expectedCreatorShare = 750_000;
        uint256 expectedAttributionShare = 250_000;

        // Assert that the actual values are different due to precision loss
        assertNotEq(shares[0], expectedCreatorShare);
        assertNotEq(shares[1], expectedAttributionShare);
    }
```
```javascript 
// add this contract under FeeManagerTest
contract ExposedFeeManager is FeeManager {
    constructor(address admin_, address protocolFeeReceiver_, address splitFactory_)
        FeeManager(admin_, protocolFeeReceiver_, splitFactory_)
    {}

    function exposedTransfer(address asset_, uint256 amount_, address from_, address to_)
        external
    {
        _transfer(asset_, amount_, from_, to_);
    }

    function exposedBuildSharesAndTargets(
        Target memory creator,
        Target[] memory attributions,
        uint32 revshareBps
    ) external pure returns (address[] memory targets, uint256[] memory shares) {
        return _buildSharesAndTargets(creator, attributions, revshareBps);
    }
}
```
This is a walkthrough of this test above:
1. Creates a creator target and an attributions array with a single target.
2. Sets revshareBps to 5000, representing a 50% revenue share.
3. Calls the `exposedBuildSharesAndTargets` function to get the targets and shares arrays.
4. The expected values for creatorShare and attributionShare are calculated assuming multiplication is performed before division.
5. The test asserts that the actual values in the shares array are different from the expected values due to precision loss.

## Impact
The incorrect order of arithmetic operations in `_buildSharesAndTargets` can lead to inaccurate share calculations for the creator and attributions. This means that the creator and attributions may receive incorrect amounts of shares, potentially affecting the revenue distribution among the parties involved.

## Code Snippet

This vulnerability is in the `__buildSharesAndTargets` function and can be found here: https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol?plain=1#L476

## Tool used
Foundry
Manual Review

## Recommendation
To mitigate this vulnerability, consider rearranging the arithmetic operations in the `_buildSharesAndTargets` function to perform multiplication before division.