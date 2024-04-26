Winning Scarlet Yeti

medium

# MAX_ROYALTY_BPS not used

## Summary

The entire amount (100%) of the protocol fee can be routed to the referrers, breaking an important protocol invariant that only a maximum of 95% of the protocol fee can be routed to the referrers. This leads to a loss of assets to the protocol as the protocol will end up receiving nothing.

## Vulnerability Detail

Per the comment on Lines 295-296 below, the mint referrer revenue share (`mintReferrerRevshareBps`) plus collection referrer revenue share (`collectionReferrerRevshareBps`) must not exceed `MAX_ROYALTY_BPS` (9500).

However, in Line 308 below, it was found that the implementation does not adhere to the requirement, and the mint referrer revenue share plus collection referrer revenue share cannot exceed `MAX_BPS` (10000) instead of `MAX_ROYALTY_BPS` (9500). As a result, the entire amount of the protocol fee can be routed to the referrers.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L295

```solidity
File: FeeManager.sol
291:     /// @notice Updates the protocol fees which are collected for various actions.
292:     /// @param protocolCreationFee_ The new protocol creation fee. This fee is collected when a new {Edition} is created. Cannot exceed {MAX_PROTOCOL_FEE}.
293:     /// @param protocolFlatFee_ The new protocol flat fee. This fee is collected on all mint transactions. Cannot exceed {MAX_PROTOCOL_FEE}.
294:     /// @param protocolFeeShareBps_ The new protocol fee share in basis points. Cannot exceed {MAX_PROTOCOL_FEE_BPS}.
295:     /// @param mintReferrerRevshareBps_ The new mint referrer revenue share in basis points. This plus the collection referrer share cannot exceed {MAX_ROYALTY_BPS}.
296:     /// @param collectionReferrerRevshareBps_ The new collection referrer revenue share in basis points. This plus the mint referrer share cannot exceed {MAX_ROYALTY_BPS}.
297:     /// @dev This function can only be called by the owner or an admin.
298:     function setProtocolFees(
299:         uint64 protocolCreationFee_,
300:         uint64 protocolFlatFee_,
301:         uint16 protocolFeeShareBps_,
302:         uint16 mintReferrerRevshareBps_,
303:         uint16 collectionReferrerRevshareBps_
304:     ) external onlyOwnerOrRoles(ADMIN_ROLE) {
305:         if (
306:             protocolCreationFee_ > MAX_PROTOCOL_FEE || protocolFlatFee_ > MAX_PROTOCOL_FEE
307:                 || protocolFeeShareBps_ > MAX_PROTOCOL_FEE_BPS
308:                 || (mintReferrerRevshareBps_ + collectionReferrerRevshareBps_) > MAX_BPS
309:         ) {
310:             revert InvalidFee();
311:         }
312:         protocolCreationFee = protocolCreationFee_;
313:         protocolFlatFee = protocolFlatFee_;
314:         protocolFeeshareBps = protocolFeeShareBps_;
315:         mintReferrerRevshareBps = mintReferrerRevshareBps_;
316:         collectionReferrerRevshareBps = collectionReferrerRevshareBps_;
317:     }
```

## Impact

The entire amount (100%) of the protocol fee can be routed to the referrers, breaking an important protocol invariant that only a maximum of 95% of the protocol fee can be routed to the referrers. This leads to a loss of assets to the protocol as the protocol will end up receiving nothing.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L295

## Tool used

Manual Review

## Recommendation

Consider making the following changes:

```diff
function setProtocolFees(
    uint64 protocolCreationFee_,
    uint64 protocolFlatFee_,
    uint16 protocolFeeShareBps_,
    uint16 mintReferrerRevshareBps_,
    uint16 collectionReferrerRevshareBps_
) external onlyOwnerOrRoles(ADMIN_ROLE) {
    if (
        protocolCreationFee_ > MAX_PROTOCOL_FEE || protocolFlatFee_ > MAX_PROTOCOL_FEE
            || protocolFeeShareBps_ > MAX_PROTOCOL_FEE_BPS
-            || (mintReferrerRevshareBps_ + collectionReferrerRevshareBps_) > MAX_BPS
+            || (mintReferrerRevshareBps_ + collectionReferrerRevshareBps_) > MAX_ROYALTY_BPS
    ) {
        revert InvalidFee();
    }
```