Winning Scarlet Yeti

high

# Collection referral minting fee will be burned if batch minting feature is used

## Summary

If the batch minting feature is used, the collection referral minting fee will be burned. This will result in a loss of the collection referral minting fee for the affected referrers, as the fee will be burned instead of being routed to the designated collection referrer.

## Vulnerability Detail

When the batch minting feature is used, the referrer will be hardcoded to zero, as shown in Line 288 and Line 312 below.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L288

```solidity
File: Edition.sol
277:     function mintBatch(
278:         address to_,
279:         uint256[] calldata tokenIds_,
280:         uint256[] calldata amounts_,
281:         bytes calldata data_
282:     ) external payable {
283:         for (uint256 i = 0; i < tokenIds_.length; i++) {
284:             Work storage work = works[tokenIds_[i]];
285: 
286:             // wake-disable-next-line reentrancy
287:             FEE_MANAGER.collectMintFee{value: msg.value}(
288:                 this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy // @audit-info payer_ => msg.sender, referrer_ => address(0)
289:             );
..SNIP..
297:     }
```

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L312

```solidity
304:     function mintBatch(
305:         address[] calldata receivers_,
306:         uint256 tokenId_,
307:         uint256 amount_,
308:         bytes calldata data_
309:     ) external payable {
310:         // wake-disable-next-line reentrancy
311:         FEE_MANAGER.collectMintFee{value: msg.value}(
312:             this, tokenId_, amount_, msg.sender, address(0), works[tokenId_].strategy // @audit-info payer_ => msg.sender, referrer_ => address(0)
313:         );
..SNIP..
320:     }
```

Note that the `referrer_` is hardcoded to zero in the earlier step. At Lines 436 below,  the collection referral fee (`collectionReferrerShare`) will be collected from the payer and forwarded to address zero, effectively burning the collection referral minting fee instead of routing it to the designated collection referrer.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L436

```solidity
File: FeeManager.sol
412:     function _splitProtocolFee(
413:         IEdition edition_,
414:         address asset_,
415:         uint256 amount_,
416:         address payer_,
417:         address referrer_
418:     ) internal returns (uint256 referrerShare) {
419:         // The creation and mint referrers earn 25% and 50% of the protocol's share respectively, if applicable
420:         uint256 mintReferrerShare = getMintReferrerShare(amount_, referrer_);
421:         uint256 collectionReferrerShare = getCollectionReferrerShare(amount_, referrers[edition_]);
422:         referrerShare = mintReferrerShare + collectionReferrerShare;
..SNIP..
430:         _route(
431:             Fee({asset: asset_, amount: mintReferrerShare}),
432:             Target({target: referrer_, chainId: block.chainid}),
433:             payer_
434:         );
435: 
436:         _route(
437:             Fee({asset: asset_, amount: collectionReferrerShare}),
438:             Target({target: referrer_, chainId: block.chainid}),
439:             payer_
440:         );
```

## Impact

Loss of referral minting fee as they will be burned instead of routed to the designated collection referrer.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L288

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L312

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L436

## Tool used

Manual Review

## Recommendation

Consider the following change to ensure that the collection referral fee is not burned when the batch minting feature is used.

```diff
function _splitProtocolFee(
    IEdition edition_,
    address asset_,
    uint256 amount_,
    address payer_,
    address referrer_
) internal returns (uint256 referrerShare) {
    // The creation and mint referrers earn 25% and 50% of the protocol's share respectively, if applicable
    uint256 mintReferrerShare = getMintReferrerShare(amount_, referrer_);
    uint256 collectionReferrerShare = getCollectionReferrerShare(amount_, referrers[edition_]);
    referrerShare = mintReferrerShare + collectionReferrerShare;
		..SNIP..
    _route(
        Fee({asset: asset_, amount: mintReferrerShare}),
        Target({target: referrer_, chainId: block.chainid}),
        payer_
    );

    _route(
        Fee({asset: asset_, amount: collectionReferrerShare}),
-       Target({target: referrer_, chainId: block.chainid}),
+				Target({target: referrers[edition_], chainId: block.chainid}),
        payer_
    );
}
```