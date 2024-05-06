Winning Scarlet Yeti

high

# Collection referrers will not receive their share of the minting fee

## Summary

Collection referrers will not receive their share of the minting fee, leading to a loss of assets for the collection referrers.

## Vulnerability Detail

> [!NOTE]
>
> The following information is taken from the sponsor's response in the Contest's Discord Channel.
>
> There are two types of referrers in this ecosystem. The first is a collection referrer, who refers the creation of a new Edition, and gets a cut of all mint fees for that Edition. The second is a mint referrer, who refers a mint and gets a cut of the mint fee for that mint.
>
> The following describes the difference between collection referrer and mint referrer.
> 
> - User 1 creates a referral link to create a collection
> - User 2 uses that link to publish a collection
> - User 3 mints that collection. For this mint, user1 is treated as the collection referrer
>- User 4 creates a referral link to mint that collection
> - User 5 mints that collection. For this mint, user1 is treated as the collection referrer and user4 is treated as the mint referrer

Assume that Alice creates a referral link to create a collection. Bob uses that link to publish a collection called $Collection_1$. In this case, the collection referrer of $Collection_1$​ will be set to Alice.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L103

```solidity
File: TitlesCore.sol
098:     /// @notice Publishes a new Work in the given {Edition} using the given payload.
099:     /// @param edition_ The {Edition} to publish the Work in.
100:     /// @param payload_ The compressed payload for publishing the Work. See {WorkPayload}.
101:     /// @param referrer_ The address of the referrer.
102:     /// @return tokenId The token ID of the new Work.
103:     function publish(Edition edition_, bytes calldata payload_, address referrer_)
..SNIP..
140:         // Create the fee route for the new Work
141:         // wake-disable-next-line reentrancy
142:         Target memory feeReceiver = feeManager.createRoute(
143:             edition_, tokenId, _attributionTargets(work_.attributions), referrer_
..SNIP..
```

When the `TitlesCore.publish` is executed to create a new collection, the `feeManager.createRoute` function will be executed internally. The `feeManager.createRoute` function will store Alice's wallet address within the `referrers[edition_]` mapping. Thus, the state of the `referrers` mapping will be as follows:

```solidity
referrers[Edition_A] = Alice
```

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L125

```solidity
File: FeeManager.sol
125:     function createRoute(
126:         IEdition edition_,
127:         uint256 tokenId_,
128:         Target[] calldata attributions_,
129:         address referrer_
130:     ) external onlyOwnerOrRoles(ADMIN_ROLE) returns (Target memory receiver) {
..SNIP..
158:         _feeReceivers[getRouteId(edition_, tokenId_)] = receiver;
159:         referrers[edition_] = referrer_;
160:     }
```

When someone mints a new token for $Collection_A$​, Alice, who is the collection referral, should get a share of the minting fee. However, based on the current implementation of the `FeeManager._splitProtocolFee` function below, Alice will not receive her collection referral fee. Instead, the collection referral fee is routed to the mint referrer.

Line 438 below shows that the collection referral fee is routed to the mint referrer (`referrer_`) instead of the collection referrer (`referrers[edition_]`).

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L412

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
420:         uint256 mintReferrerShare = getMintReferrerShare(amount_, referrer_); // @audit-info mint referrer
421:         uint256 collectionReferrerShare = getCollectionReferrerShare(amount_, referrers[edition_]); // @audit-info collection referrer 
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
441:     }

```

## Impact

Following are some of the negative impacts that could lead to a loss of assets:

1. Collection referrer will not receive their share of the minting fee.
2. Malicious minter of the tokens can set the `referrer_` parameter to their own wallet address when executing the `Edition.mint` function, so that the collection referral fee can be routed to their own wallet, effectively avoiding paying the collection referral fee to someone else. 

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L103

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L125

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L412

## Tool used

Manual Review

## Recommendation

Consider the following change to ensure that the collection referral fee is routed to the collection referrer instead of the mint referrer.

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
        Fee({asset: asset_, amount: collectionReferrerShare}),
-        Target({target: referrer_, chainId: block.chainid}),
+        Target({target: referrers[edition_], chainId: block.chainid}),
        payer_
    );
}
```