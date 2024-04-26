Winning Scarlet Yeti

high

# Incorrect assets will be charged during minting and creation of work.

## Summary

Incorrect assets will be charged as the minting fee or creation fee, resulting in a loss of assets for either the users or the creator.

## Vulnerability Detail

During the publishing of a new work/collection of an Edition, the creator is given the option to set the fee to be collected to ETH or an arbitrary ERC20 token, as shown in Line 337 below. Assume that Bob wants to collect the fee in WETH. Thus, he will set the `strategy_.asset` to USDC ERC20 address when creating his new work/collection.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L337

```solidity
File: FeeManager.sol
322:     function validateStrategy(Strategy calldata strategy_)
323:         external
324:         pure
325:         returns (Strategy memory strategy)
326:     {
327:         // Clamp the revshare to the range of [MIN_ROYALTY_BPS...MAX_ROYALTY_BPS]
328:         uint16 revshareBps = strategy_.revshareBps > MAX_ROYALTY_BPS
329:             ? MAX_ROYALTY_BPS
330:             : strategy_.revshareBps < MIN_ROYALTY_BPS ? MIN_ROYALTY_BPS : strategy_.revshareBps;
331: 
332:         // Clamp the royalty to the range of [0...MAX_ROYALTY_BPS]
333:         uint16 royaltyBps =
334:             strategy_.royaltyBps > MAX_ROYALTY_BPS ? MAX_ROYALTY_BPS : strategy_.royaltyBps;
335: 
336:         strategy = Strategy({
337:             asset: strategy_.asset == address(0) ? ETH_ADDRESS : strategy_.asset,
338:             mintFee: strategy_.mintFee,
339:             revshareBps: revshareBps,
340:             royaltyBps: royaltyBps
341:         });
342:     }
```

The `getMintFee` function will be triggered to compute the minting fee when someone mints a token from Bob's collection.

However, as shown in Line 256 below, the assets to be collected as fee is always hardcoded to native ETH (ETH_ADDRESS). Thus, ETH instead of USDC will be charged as a fee, which is incorrect.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L256

```solidity
File: FeeManager.sol
250:     function getMintFee(Strategy memory strategy_, uint256 quantity_)
251:         public
252:         view
253:         returns (Fee memory fee)
254:     {
255:         // Return the total fee (creator's mint fee + protocol flat fee)
256:         return Fee({asset: ETH_ADDRESS, amount: quantity_ * (strategy_.mintFee + protocolFlatFee)});
257:     }
```
The same issue also occurs during the creation of the new Edition. The scenario is exactly the same, and omitted for brevity.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L225

```solidity
File: FeeManager.sol
222:     /// @notice Calculates the fee for creating a new {Edition}.
223:     /// @return fee The {Fee} for creating a new {Edition}.
224:     /// @dev The creation fee is a flat fee collected by the protocol when a new {Edition} is created.
225:     function getCreationFee() public view returns (Fee memory fee) {
226:         return Fee({asset: ETH_ADDRESS, amount: protocolCreationFee});
227:     }
```

## Impact

Incorrect assets will be charged as the minting fee or creation fee, resulting in a loss of assets for either the users or the creator.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L337

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L256

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L225

## Tool used

Manual Review

## Recommendation

Consider the following change to ensure that the correct asset is charged as the minting or creation fee.

```diff
function getMintFee(Strategy memory strategy_, uint256 quantity_)
    public
    view
    returns (Fee memory fee)
{
    // Return the total fee (creator's mint fee + protocol flat fee)
-   return Fee({asset: ETH_ADDRESS, amount: quantity_ * (strategy_.mintFee + protocolFlatFee)});
+   return Fee({asset: strategy_.asset, amount: quantity_ * (strategy_.mintFee + protocolFlatFee)});
}
```