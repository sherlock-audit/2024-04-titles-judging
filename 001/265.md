Winning Scarlet Yeti

high

# Original collection referrer will be overwritten when a new collection/work is created

## Summary

Original collection referrers will be overwritten when a new collection/work is created. This results in the original collection referrers being unable to collect the fee they are entitled to, leading to a loss of assets for them.

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
> - User 4 creates a referral link to mint that collection
> - User 5 mints that collection. For this mint, user1 is treated as the collection referrer and user4 is treated as the mint referrer
>

Assume that Alice creates a referral link to create a collection. Bob uses that link to publish a new collection/work called $Collection_A$ that is based on an Edition called $Edition_A$. The collection referrer of $Collection_A$ will Alice. 

Bob will call the following `TitlesCore.publish` function with the `edition` parameter set to $Edition_A$​ and the `referrer_` parameter will be automatically set to Alice by the front end.

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

Within the `TitlesCore.publish`, the following `feeManager.createRoute` function will be executed internally. The `feeManager.createRoute` function will store Alice's wallet address within the `referrers[edition_]` mapping. Thus, the state of the `referrers` mapping will be as follows:

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

When someone mints a new token for $Collection_A$​, Alice, who is the collection referrer, will get a share of the minting fee per Line 421 below.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L421

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
```

Let's assume Charles also creates a referral link to create a collection. David uses that link to publish a new collection/work called $Collection_B$ that is based on the same Edition called $Edition_A$. The collection referral of $Collection_B$ will be Charles.

When the `FeeManager.createRoute` function is executed during the publishing of the new work/collection, Charles's wallet address will be stored within the `referrers[edition_]` mapping. Thus, the state of the `referrers` mapping will be as follows:

```solidity
referrers[Edition_A] = Charles
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

It is important to note that the `referrers[Edition_A]` have been updated from Alice to Charles here. At this point onwards, if someone mints tokens for $Collection_A$, the collection referral fee will be routed to Charles instead of Alice. Alice is the referral for $Collection_A$, yet she does not receive the referral fee, resulting in a loss of assets for Alice.

## Impact

Loss of assets as shown in the above scenario. The original collection referrers are unable to collect the fee they are entitled to, leading to a loss of assets for them.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L103

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L125

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L421

## Tool used

Manual Review

## Recommendation

Consider the following changes to ensure that the collection referral fee is routed to the correct collection referrer for each collection/work. 

With the following changes, Alice will continue to receive the collection referral fee for $Collection_A$ and Bob will receive the collection referral fee for $Collection_B$ even if there are multiple collections/works for a specific Edition.

```diff
function createRoute(
    IEdition edition_,
    uint256 tokenId_,
    Target[] calldata attributions_,
    address referrer_
) external onlyOwnerOrRoles(ADMIN_ROLE) returns (Target memory receiver) {
..SNIP..
    _feeReceivers[getRouteId(edition_, tokenId_)] = receiver;
-    referrers[edition_] = referrer_;
+    referrers[getRouteId(edition_, tokenId_)] = referrer_;
}
```

```diff
function _splitProtocolFee(
    IEdition edition_,
+		uint256 tokenId,    
    address asset_,
    uint256 amount_,
    address payer_,
    address referrer_
) internal returns (uint256 referrerShare) {
    // The creation and mint referrers earn 25% and 50% of the protocol's share respectively, if applicable
    uint256 mintReferrerShare = getMintReferrerShare(amount_, referrer_);
-    uint256 collectionReferrerShare = getCollectionReferrerShare(amount_, referrers[edition_]);
+    uint256 collectionReferrerShare = getCollectionReferrerShare(amount_, referrers[getRouteId(edition_, tokenId)]);
    referrerShare = mintReferrerShare + collectionReferrerShare;
```