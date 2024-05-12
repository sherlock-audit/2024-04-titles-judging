# Issue H-1: `TitlesCore._publish()` can be DOSd by consuming the user's allowance by frontrunning `FeeManager.collectCreationFee()` 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/251 

## Found by 
ironside, threadmodeling, xiaoming90
## Summary

When using an ERC20 for fees, the `PUBLISHER_ROLE`'s allowance given to `FeeManager` or the one of the user calling `createEdition()` can be freely consumed through `FeeManager.collectCreationFee()`. This will move funds to the `FeeManager`'s `protocolFeeReceiver` and DOS the `TitlesCore._publish()` function (which is the main functionality on the `TitlesCore` contract).

## Vulnerability Detail

In the `TitlesCore._publish()` function, `FeeManager.collectCreationFee()` is called:

```solidity
File: TitlesCore.sol
120:     function _publish(Edition edition_, WorkPayload memory work_, address referrer_)
...
138:         feeManager.collectCreationFee{value: msg.value}(edition_, tokenId, msg.sender);
```

`FeeManager.collectCreationFee()` isn't permissioned or access-controlled, therefore it can be called by anyone:

```solidity
File: FeeManager.sol
166:     function collectCreationFee(IEdition edition_, uint256 tokenId_, address feePayer_)
167:         external
168:         payable
169:     {
170:         Fee memory fee = getCreationFee();
171:         if (fee.amount == 0) return;
172: 
173:         _route(fee, Target({target: protocolFeeReceiver, chainId: block.chainid}), feePayer_);
174:         emit FeeCollected(address(edition_), tokenId_, ETH_ADDRESS, fee.amount, 0);
175:     }
```

As we can see above, if there's a `fee.amount` to pay, the receiver is fetched from storage but the `feePayer_` depends on the (malicious) user's input.

The `_route` function is basically a transfer that can either occur with native Ether or with an ERC20 token:

```solidity
File: FeeManager.sol
448:     function _route(Fee memory fee_, Target memory feeReceiver_, address feePayer_) internal {
449:         // Cross-chain fee routing is not supported yet
450:         if (block.chainid != feeReceiver_.chainId) revert NotRoutable();
451:         if (fee_.amount == 0) return;
452: 
453:         _transfer(fee_.asset, fee_.amount, feePayer_, feeReceiver_.target); 
454:     }
...
461:     function _transfer(address asset_, uint256 amount_, address from_, address to_) internal {
462:         if (asset_ == ETH_ADDRESS) {
463:             to_.safeTransferETH(amount_);
464:         } else {
465:             asset_.safeTransferFrom(from_, to_, amount_);
466:         }
467:     }
```

Given that, here, the `from` parameter is controlled by the malicious user's input, it means that the frontrunning attack will transfer `fee.amount` of `fee.asset` from `victim` to `protocolFeeReceiver`.

While the amount isn't user controlled: multiple calls to `collectCreationFee()` can be made to either drain the victim's set allowance or funds (if max approval was given), whichever gives in first.

The funds will probably be recoverable due `protocolFeeReceiver` being probably trusted to send them back, but draining the allowance will make any subsequent calls to `TitlesCore._publish()` to revert, at only the cost of gas calls for the attacker, effectively DOSing the publish functionality on `TitlesCore`

## Impact

While DOS attacks vary in severity, this one is easy and cheap for the attacker. Also, DOSing the `PUBLISHER_ROLE` and the `TitlesCore._publish()` will additionally move their funds around. Hence, this is submitted as a High Severity bug

## Code Snippet

- <https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L166-L175>
- <https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L448-L467>
- <https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L120>
- <https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L138>

## Tool used

Manual Review

## Recommendation

Seeing a `transferFrom` function with `from` being something else than `msg.sender` will often result in such a bug.
The flow of funds needs to be changed so that `FeeManager`'s functions become permissioned or that approval is given and reset in the same transaction.



## Discussion

**pqseags**

Is this only valid for ERC20 fees, or for ETH fees? If ERC20 then this shouldn't be in scope, but if it is valid for ETH as well then it is in scope. 

# Issue H-2: Users can exploit the batch minting feature to avoid paying minting fees for tokens 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/264 

## Found by 
0x73696d616f, 1337, 4rdiii, BiasedMerc, ComposableSecurity, Kalogerone, Shaheen, ast3ros, bareli, blockchain555, cducrest-brainbot, ff, funkornaut, gesha17, gladiator111, jennifer37, joicygiore, juan, sammy, techOptimizor, trachev, xiaoming90, y4y
## Summary

Users can exploit the batch minting feature to avoid paying minting fees for tokens, leading to a loss of fees for the creator and fee recipients.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L304

```solidity
File: Edition.sol
304:     function mintBatch(
305:         address[] calldata receivers_,
306:         uint256 tokenId_,
307:         uint256 amount_,
308:         bytes calldata data_
309:     ) external payable {
310:         // wake-disable-next-line reentrancy
311:         FEE_MANAGER.collectMintFee{value: msg.value}(
312:             this, tokenId_, amount_, msg.sender, address(0), works[tokenId_].strategy
313:         );
314: 
315:         for (uint256 i = 0; i < receivers_.length; i++) {
316:             _issue(receivers_[i], tokenId_, amount_, data_);
317:         }
318: 
319:         _refundExcess();
320:     }
```

Assume that the total mint fee (protocol flat fee = 0.0006 ETH + minting fee = 0.0004) for each token is 0.001 ETH. Bob wants to mint 1000 tokens, and he has to pay a minting fee of 1 ETH.

However, the issue is that Bob can mint 1000 tokens without paying the full mint fee of 1 ETH as shown in the step below:

1. Bob calls the above `mintBatch` function with `receivers_` array set to `1000 x Bob address` and the `amount_` parameter set to `1`.

2. Line 331 above will compute the mint fee that Bob needs to pay. In this case, the `amount_` is one (1), and the total mint fee will be equal to `1 X 0.001 ETH`.

3. The for-loop at Line 315 above will loop 1000 times as the `receivers_.length` is 1000. Each loop will mint one token to Bob. 
4. At the end of the for loop, Bob will receive 1000 while only paying a minting fee of 0.001 ETH instead of 1 ETH.
5. In other words, Bob only paid the minting fee for the first token and did not pay the minting fee for the rest of the 999 tokens.

## Impact

Loss of fees for the creator and fee recipients as minters can avoid paying the fee using the trick mentioned in this report.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L304

## Tool used

Manual Review

## Recommendation

Consider the following change to ensure that the minting fee is computed based on the total number of tokens minted to all receivers.

```diff
    function mintBatch(
        address[] calldata receivers_,
        uint256 tokenId_,
        uint256 amount_,
        bytes calldata data_
    ) external payable {
        // wake-disable-next-line reentrancy
        FEE_MANAGER.collectMintFee{value: msg.value}(
-            this, tokenId_, amount_, msg.sender, address(0), works[tokenId_].strategy
+            this, tokenId_, amount_ * receivers_.length, msg.sender, address(0), works[tokenId_].strategy
        );

        for (uint256 i = 0; i < receivers_.length; i++) {
            _issue(receivers_[i], tokenId_, amount_, data_);
        }

        _refundExcess();
    }
```



## Discussion

**pqseags**

Valid, will fix

# Issue H-3: Original collection referrer will be overwritten when a new collection/work is created 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/265 

## Found by 
14si2o\_Flint, BengalCatBalu, Varun\_05, trachev, xiaoming90
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



## Discussion

**pqseags**

Valid, but duplicate of #59 

# Issue H-4: Collection referrers will not receive their share of the minting fee 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/267 

## Found by 
0rpse, 0x73696d616f, AhmedAdam, BiasedMerc, FastTiger, Matin, Varun\_05, ZdravkoHr., ast3ros, blockchain555, cryptic, cu5t0mPe0, eeshenggoh, ff, gladiator111, i3arba, ironside, jennifer37, kennedy1030, merlinboii, mt030d, sammy, xiaoming90
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



## Discussion

**pqseags**

Valid, will fix

# Issue H-5: Excess ETH of the victim can be stolen by malicious external parties due to re-entrancy attack 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/268 

## Found by 
xiaoming90, zigtur
## Summary

Excess ETH of the victim can be stolen by malicious external parties due to re-entrancy attack.

## Vulnerability Detail

Per Line 241 below, it is expected that there will be excess ETH residing on the contract at the end of the transaction. The `_refundExcess` function is implemented with the intention of sweeping excess ETH back to the caller of the `mint` function at the end of the transaction.

However, the problem is when someone (e.g., Bob) mints a new token for a given work, a number of parties can gain control of the transaction and re-enter the Edition contract. The `mint` function uses a number of functions that can pass execution control to external parties.

The following is a list of external parties that could re-enter during the minting execution:

1. Fee recipients (e.g., Edition's creator and attribution, Collection and minting referrers) - At Line 236, when the FEE_MANAGER.collectMintFee is executed, native ETH is transferred to the fee recipients, who will gain control of the execution.

1. Token recipients - At Line 240 below, the `_issue` function will call the `ERC1155._mint` function internally.  The `ERC1155._mint` contains an ERC1155 hook (`onERC1155Received`) that will pass the control to the token recipient, who will gain control of the execution.

Once any of the above-mentioned parties gain control of the execution, they can re-enter the Edition contract and execute any of the mint functions (all mint functions will execute the _refundExcess function). This will result in the excess ETH residing on the contract being swept to the malicious party's address instead of Bob's address, effectively stealing Bob's assets.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L228

```solidity
File: Edition.sol
228:     function mint(
229:         address to_,
230:         uint256 tokenId_,
231:         uint256 amount_,
232:         address referrer_,
233:         bytes calldata data_
234:     ) external payable override {
235:         // wake-disable-next-line reentrancy
236:         FEE_MANAGER.collectMintFee{value: msg.value}(
237:             this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
238:         );
239: 
240:         _issue(to_, tokenId_, amount_, data_);
241:         _refundExcess();
242:     }
```

> [!IMPORTANT]
>
> This issue also affects the `Edition.mintWithComment` and `Edition.mintBatch` functions. The write-up is the same and is omitted for brevity.

## Impact

Loss of assets, as shown in the above scenario. The victim's excess ETH can be stolen by malicious external parties.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L228

## Tool used

Manual Review

## Recommendation

Consider adding a re-entrancy guard to all minting functions to prevent the fee recipient from re-entering the Edition contract during issuing/minting.

```diff
function mint(
    address to_,
    uint256 tokenId_,
    uint256 amount_,
    address referrer_,
    bytes calldata data_
- ) external payable override {
+ ) external payable override nonReentrant() {
```



## Discussion

**pqseags**

Will address

# Issue H-6: Excess ETH will be stuck in the Fee Manager contract and not swept back to the users 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/269 

## Found by 
BiasedMerc, ZdravkoHr., joicygiore, kn0t, techOptimizor, valentin2304, xiaoming90, y4y, zigtur, zoyi
## Summary

Excess ETH will be stuck in the Fee Manager contract and not swept back to the users.

## Vulnerability Detail

Per Line 241 below, it is expected that there will be excess ETH residing on the contract at the end of the transaction. The `_refundExcess` function is implemented with the intention of sweeping excess ETH back to the caller of the `mint` function at the end of the transaction.

Assume Bob transfers 0.05 ETH to the Edition contract, but the minting fee ends up being only 0.03 ETH. The _refundExcess function at the end of the function (Line 241 below) is expected to return the excess 0.02 ETH back to Bob.

However, it was found that such a design does not work. When the `collectMintFee` function is executed on Line 236 below, the entire amount of ETH (0.05 ETH) will be forwarded to the Fee Manager contract. 0.03 ETH out of 0.05 ETH will be forwarded to the fee recipients, while the remaining 0.02 will be stuck in the Fee Manager contract. The excess 0.02 is not being returned to the Edition contract. Thus, when the `_refundExcess` function is triggered at the end of the function, no ETH will be returned to Bob.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L228

```solidity
File: Edition.sol
228:     function mint(
229:         address to_,
230:         uint256 tokenId_,
231:         uint256 amount_,
232:         address referrer_,
233:         bytes calldata data_
234:     ) external payable override {
235:         // wake-disable-next-line reentrancy
236:         FEE_MANAGER.collectMintFee{value: msg.value}(
237:             this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
238:         );
239: 
240:         _issue(to_, tokenId_, amount_, data_);
241:         _refundExcess();
242:     }
```

> [!IMPORTANT]
>
> This issue also affects the `Edition.mintWithComment` and `Edition.mintBatch` functions. The write-up is the same and is omitted for brevity.

In addition, the Contest's README mentioned that the protocol aims to aims to avoid any direct TVL in this release:

> Please discuss any design choices you made.
>
> Fund Management: We chose to delegate fee payouts to 0xSplits v2. The protocol aims to avoid any direct TVL in this release.

In other words, this means that no assets should be locked within the protocol. However, as shown in the earlier scenario, some assets will be stored in the Fee Manager, breaking this requirement.

## Impact

Excess ETH will be stuck in the Fee Manager contract and not swept back to the users.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L228

## Tool used

Manual Review

## Recommendation

Consider forwarding only the required amount of the minting fee to the Fee Manager, so that any excess ETH can be sweeped by the `_refundExcess()` function at the end of the transaction.

```diff
function mint(
    address to_,
    uint256 tokenId_,
    uint256 tokenId_,
    address referrer_,
    bytes calldata data_
) external payable override {
+		uint256 mintFee = FEE_MANAGER.getMintFee(this, tokenId_, amount_);

    // wake-disable-next-line reentrancy
-   FEE_MANAGER.collectMintFee{value: msg.value}(
+   FEE_MANAGER.collectMintFee{value: mintFee}(
        this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
    );

    _issue(to_, tokenId_, amount_, data_);
    _refundExcess();
}
```



## Discussion

**pqseags**

Will address

# Issue M-1: Incorrect encoding of bytes for EIP712 digest in `TitleGraph` causes signatures generated by common EIP712 tools to be unusable 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/74 

## Found by 
0x73696d616f, T1MOH, ZanyBonzy, ast3ros, fugazzi, mt030d
## Summary

The signature in `﻿TitleGraph.acknowledgeEdge()` and ﻿`TitleGraph.unacknowledgeEdge()` is generated based on a digest computed from ﻿`edgeId` and ﻿`data`. However, the ﻿`data` bytes argument is not correctly encoded according to the EIP712 specification. Consequently, a signature generated using common EIP712 tools would not pass validation in ﻿`TitleGraph.checkSignature()`.

## Vulnerability Detail
According to [EIP712](https://eips.ethereum.org/EIPS/eip-712#definition-of-encodedata):
> The dynamic values bytes and string are encoded as a keccak256 hash of their contents.

```solidity
    modifier checkSignature(bytes32 edgeId, bytes calldata data, bytes calldata signature) {
        bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
        ...
    }
```
However, the `checkSignature()` modifier in the `TitlesGraph` contract reconstructs the digest by encoding the ﻿data bytes argument without first applying keccak256 hashing.
As a result, a signature generated using common EIP712 tools (e.g. using the `signTypedData` function from `ethers.js`) would not pass validation in ﻿`TitleGraph.checkSignature()`.

### POC
1. EIP712 signature computed by using ethers.js
```js
// main.js
const { ethers } = require("ethers");

async function main() {
    const pk = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80";
    const signer = new ethers.Wallet(pk);
    const domain = {
      name: "TitlesGraph",
      version: '1',
      chainId: 31337,
      verifyingContract: "0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496" // should match the address in foundry test
    };
    const types = {
      Ack: [
        { name: "edgeId", type: "bytes32" },
        { name: "data", type: "bytes" },
      ],
    };
    const value = {
        edgeId: ethers.id("test edgeId"),
        data: "0xabcd"
    };
    const signature = await signer.signTypedData(domain, types, value);
    console.log(signature);
    
}

main();
```
here we run 
```bash
npm install ethers
node main.js
```
The output is `0xab4623a7bacf25ed3d6779684f195ed63a5ed1ed46c278c107390086e74b739b35f1db213c6075dedc041d68ced3d11798d49afaf3c47743d4696c49f03037b51b`

2. EIP712 signature computed using foundry
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {EIP712} from "lib/solady/src/utils/EIP712.sol";

contract EIP712Test is Test, EIP712 {
    bytes32 public constant ACK_TYPEHASH = keccak256("Ack(bytes32 edgeId,bytes data)");

    // test data
    bytes32 testEdgeId = keccak256("test edgeId");
    bytes testData = hex"abcd";


    function test_sig() public {
        uint256 pk = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
        bytes32 digest = _computeDigest(testEdgeId, testData);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(pk, digest);
        bytes memory signature = abi.encodePacked(r, s, v);
        console.logBytes(signature);
    }

    function test_sigShouldBe() public {
        uint256 pk = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
        bytes32 digest = _computeDigestShouldBe(testEdgeId, testData);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(pk, digest);
        bytes memory signature = abi.encodePacked(r, s, v);
        console.logBytes(signature);
    }

    function _computeDigest(bytes32 edgeId, bytes memory data) internal returns (bytes32 digest) {
        digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
    }

    function _computeDigestShouldBe(bytes32 edgeId, bytes memory data) internal returns (bytes32 digest) {
        digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, keccak256(data))));
    }

    function _domainNameAndVersion()
        internal
        pure
        override
        returns (string memory name, string memory version)
    {
        name = "TitlesGraph";
        version = "1";
    }
}
```
here we run
```bash
forge test --mc EIP712Test -vv
```
The output is
```txt
[PASS] test_sig() (gas: 12176)
Logs:
  0x7bd09aece710ef3845f26c4a695d357b1b170f75d0702f18ec09409f571260237a38e0fed802f8a9d598d9aed0d7898562c51e09bfa7cf254e5a8a5bc74106561c

[PASS] test_sigShouldBe() (gas: 11958)
Logs:
  0xab4623a7bacf25ed3d6779684f195ed63a5ed1ed46c278c107390086e74b739b35f1db213c6075dedc041d68ced3d11798d49afaf3c47743d4696c49f03037b51b
```

`test_sig()` simulates the way the digest is reconstructed in `TitleGraph.checkSignature()`, while `test_sigShouldBe()` shows how  the digest should be reconstructed.
From the above output, we can see the signature generated by ethers.js matches the signature generated in `test_sigShouldBe()`  and does not match the signature generated in `test_sig()`.
This PoC shows the way `TitleGraph.checkSignature()` reconstruct the digest is not compatible with the way data is encoded in EIP712.

## Impact
A signature generated by the signer using common EIP712 tools (e.g. signTypedData in `ethers.js`) would not pass validation in ﻿`TitleGraph.checkSignature()`.

## Code Snippet
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L41

## Tool used

Manual Review, ethers.js, foundry

## Recommendation
Encoding the `data` bytes as a keccak256 hash of its contents before computing the digest from it:
```diff
- digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
+ digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, keccak256(data))));
```

# Issue M-2: ERC2981 royalties discrepancy with strategy 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/144 

## Found by 
ComposableSecurity, cducrest-brainbot, sammy
## Summary

In `Edition.sol`, functions that set the value of `works[tokenId].strategy` which includes `works[tokenId].strategy.royaltyBps` do not set ERC2981's internal token royalty value.

## Vulnerability Detail

The function `setFeeStrategy()` sets the public mapping value `works[tokenId_].strategy` which may update the `roylatyBps` value but does not call `_setTokenRoyalty(...)`:

```solidity
    function setFeeStrategy(uint256 tokenId_, Strategy calldata strategy_) external {
        if (msg.sender != works[tokenId_].creator) revert Unauthorized();
        works[tokenId_].strategy = FEE_MANAGER.validateStrategy(strategy_);  // @audit does not set royalties
    }
```

Similarly, the `publish()` function to create a new work sets the strategy but does not call `_setTokenRoyalty()`:

```solidity
    function publish(
        address creator_,
        uint256 maxSupply_,
        uint64 opensAt_,
        uint64 closesAt_,
        Node[] calldata attributions_,
        Strategy calldata strategy_,
        Metadata calldata metadata_
    ) external override onlyRoles(EDITION_MANAGER_ROLE) returns (uint256 tokenId) {
        tokenId = ++totalWorks;

        _metadata[tokenId] = metadata_;
        works[tokenId] = Work({
            creator: creator_,
            totalSupply: 0,
            maxSupply: maxSupply_,
            opensAt: opensAt_,
            closesAt: closesAt_,
            strategy: FEE_MANAGER.validateStrategy(strategy_)
        });

        Node memory _node = node(tokenId);
        for (uint256 i = 0; i < attributions_.length; i++) {
            // wake-disable-next-line reentrancy, unchecked-return-value
            GRAPH.createEdge(_node, attributions_[i], attributions_[i].data);
        }

        emit Published(address(this), tokenId);
    } 
```

This latter case may be less of a problem since `TitlesCore` calls `edition_.setRoyaltyTarget()` right after publishing a new work. However it remains a problem if publishers are expected to interact directly with `Edition` and not only through `TitlesCore`

## Impact

The value returned by `works[tokenId].strategy.royaltyBps` and `ERC2981.royaltyInfo(tokenId, salePrice)` will not be coherent. Users may expect to set a certain royalty bps while the value is not updated. Core values used for royalty payments may become incorrect after updates.

## Code Snippet

https://github.com/vectorized/solady/blob/main/src/tokens/ERC2981.sol

https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/editions/Edition.sol#L368-L371

https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/editions/Edition.sol#L121

## Tool used

Manual Review

## Recommendation

Call `_setTokenRoyalty(tokenId, FeeManager.feeReceiver(address(this), tokenId), works[tokenId].strategy.royaltyBps);` at the end of `setFeeStrategy()`. 



## Discussion

**pqseags**

This is valid and should be fixed

# Issue M-3: [M-1] Precision loss due to division before multiplication in `FeeManager._buildSharesAndTargets` (Incorrect Order of Operations) 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/201 

## Found by 
Bigsam, CodeWasp, Squilliam
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



## Discussion

**pqseags**

Will fix

# Issue M-4: TitlesGraph::acknowledgeEdge() methods do not write acknowledgments to storage 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/212 

## Found by 
4rdiii, AlexCzm, CodeWasp, KupiaSec, alexzoid, brakeless, cducrest-brainbot, den\_sosnovskyi, fugazzi, i3arba, ironside, kennedy1030, radin200, recursiveEth, ubl4nk
## Summary
When `acknowledgeEdge()` is called, the downstream call to the `_setAcknowledged()` method caches `edges[edgeId_]` in memory, instead of storage, which does not preserve changes to the `Edge` struct after the transaction concludes. 

## Vulnerability Detail
```solidity
function acknowledgeEdge(bytes32 edgeId_, bytes calldata data_)
    external
    override
    returns (Edge memory edge)
{
    if (!_isCreatorOrEntity(edges[edgeId_].to, msg.sender)) revert Unauthorized();
    return _setAcknowledged(edgeId_, data_, true);
}

function _setAcknowledged(bytes32 edgeId_, bytes calldata data_, bool acknowledged_)
    internal
    returns (Edge memory edge)
{
    if (!_edgeIds.contains(edgeId_)) revert NotFound();
    edge = edges[edgeId_];
    edge.acknowledged = acknowledged_;    // @audit does this actually write to storage? the state isn't saved? 

    if (acknowledged_) {
        emit EdgeAcknowledged(edge, msg.sender, data_);
    } else {
        emit EdgeUnacknowledged(edge, msg.sender, data_);
    }
}
```

In the code snippet above, `edges[edgeId_]` is cached in `edge`, then modified.

The issue is that `edge`, the return variable, is marked as memory, which does not save the state after the transaction ends.

Therefore, the `acknowledged` value never changes and the `acknowledgeEdge()` methods do not work as intended.

Add the following test to `TitlesGraph.t.sol`. 

Run with the following command: `forge test --match-test test_acknowledgeEdgeFailure -vvvv`
```solidity
function test_acknowledgeEdgeFailure() public {
    Node memory from = Node({
        nodeType: NodeType.COLLECTION_ERC1155,
        entity: Target({target: address(1), chainId: block.chainid}),
        creator: Target({target: address(2), chainId: block.chainid}),
        data: ""
    });

    Node memory to = Node({
        nodeType: NodeType.TOKEN_ERC1155,
        entity: Target({target: address(3), chainId: block.chainid}),
        creator: Target({target: address(4), chainId: block.chainid}),
        data: abi.encode(42)
    });

    // Only the `from` node's entity can create the edge.
    vm.prank(from.entity.target);
    titlesGraph.createEdge(from, to, "");

    vm.expectEmit(true, true, true, true);
    emit IEdgeManager.EdgeAcknowledged(
        Edge({from: from, to: to, acknowledged: true, data: ""}), to.creator.target, ""
    );

    // Only the `to` node's creator (or the entity itself) can acknowledge it
    vm.prank(to.creator.target);
    titlesGraph.acknowledgeEdge(keccak256(abi.encode(from, to)), "");

    (Node memory nodeTo,
        Node memory nodeFrom,
        bool ack,
        bytes memory dataResult
    ) = titlesGraph.edges(titlesGraph.getEdgeId(from, to));
    console.log(ack);

    // edge[edgeId].acknowledged should be set to true after successful call to titlesGraph.acknowledgeEdge()
    // However, the value is still false.
    assert(true == ack);
}
```

```solidity
[FAIL. Reason: panic: assertion failed (0x01)] test_acknowledgeEdgeFailure() (gas: 429897)
Logs:
  false
```

## Impact
Edges are unable to be acknowledged.

## Code Snippet
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L103-L124

## Tool used
Manual Review

## Recommendation
The simplest fix would be to change `memory` to `storage`.
```solidity
function _setAcknowledged(bytes32 edgeId_, bytes calldata data_, bool acknowledged_)
     internal
-    returns (Edge memory edge)
+    returns (Edge storage edge) 
 {
     if (!_edgeIds.contains(edgeId_)) revert NotFound();
     edge = edges[edgeId_];
     edge.acknowledged = acknowledged_;    

     if (acknowledged_) {
         emit EdgeAcknowledged(edge, msg.sender, data_);
     } else {
         emit EdgeUnacknowledged(edge, msg.sender, data_);
     }
 }
```




## Discussion

**pqseags**

This will be fixed

# Issue M-5: Uninitialized `TitlesCore` implementation contract can be taken over by an attacker 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/272 

## Found by 
0x73696d616f, 1337, PratRed, ZdravkoHr., alexzoid, blackhole, fibonacci, fugazzi, jecikpo, me\_na0mi, techOptimizor, threadmodeling, trachev, xiaoming90
## Summary

An attacker could take over the implementation/logic contract of  `TitlesCore`, which might impact the proxy.

## Vulnerability Detail

Per Line 21, the `TitlesCore` contract inherits from the `UUPSUpgradeable` contract. The `TitlesCore` contract will contain the logic/implementation, and the UUPS proxy will point its implementation to the `TitlesCore` contract address.

It was observed that the `TitlesCore` implementation/logic contract is left uninitialized. As a result, an attacker could take over the implementation/logic contract of  `TitlesCore` by calling the `TitlesCore.initialize` function directly on the `TitlesCore` implementation/logic contract, which might impact the proxy.

When that happens, the attackers will become the owner of the implementation/logic contract per Line 45 below.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L45

```solidity
File: TitlesCore.sol
32: contract TitlesCore is OwnableRoles, Initializable, UUPSUpgradeable, Receiver {
..SNIP..
44:     function initialize(address feeReceiver_, address splitFactory_) external initializer {
45:         _initializeOwner(msg.sender);
46: 
47:         feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
48:         graph = new TitlesGraph(address(this), msg.sender);
49:     }
```

## Impact

An attacker could take over the implementation/logic contract of  `TitlesCore`, which might impact the proxy.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L45

## Tool used

Manual Review

## Recommendation

To prevent the implementation contract from being used or taken over, invoke the `initializer` in the constructor to automatically lock the `initializer` on the implementation contract when it is deployed. This is also the recommendation from OpenZeppelin when handling upgradable contracts (https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#initializing_the_implementation_contract).

```diff
contract TitlesCore is OwnableRoles, Initializable, UUPSUpgradeable, Receiver {
    using LibClone for address;
    using LibZip for bytes;
    using SafeTransferLib for address;

    address public editionImplementation = address(new Edition());
    FeeManager public feeManager;
    TitlesGraph public graph;

+		constructor() initializer {}

    /// @notice Initializes the protocol.
    /// @param feeReceiver_ The address to receive fees.
    /// @param splitFactory_ The address of the split factory.
    function initialize(address feeReceiver_, address splitFactory_) external initializer {
        _initializeOwner(msg.sender);

        feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
        graph = new TitlesGraph(address(this), msg.sender);
    }
```

Ensure that this change is also applied to the `TitlesGraph` contract, as it is also an upgradable contract.



## Discussion

**pqseags**

Will address

# Issue M-6: Malicious users can block creators from acknowledging or deacknowledging an edge 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/273 

## Found by 
0x73696d616f, ComposableSecurity, Krace, KupiaSec, T1MOH, Yu3H0, ZdravkoHr., cryptic, fugazzi, jecikpo, jennifer37, juan, mt030d, xiaoming90, zigtur, zoyi
## Summary

Malicious users can block someone from acknowledging or deacknowledging an edge, affecting the sanctity of the data in the Graph.

## Vulnerability Detail

> [!IMPORTANT]
>
> The following is an extract from the [contest's README](https://audits.sherlock.xyz/contests/326):
>
> > Additional audit information.
> >
> > In addition to the security of funds, we would also like there to be focus on the sanctity of the data in the TitlesGraph and the permissioning around it (only the appropriate people/contracts can signal reference and acknowledgement of reference).
>
> The contest's README stated that apart from the loss of assets, the protocol team would like there to be a focus on the sanctity of the data. Thus, any issues related to the sanctity of the data in the Graph would be considered a valid Medium issue in the context of this audit contest, as the Contest's README supersede Sherlock's judging rules per Sherlock's Hierarchy of truth.

Both `acknowledgeEdge` and `unacknowledgeEdge` functions rely on the same modifier (`checkSignature`) to verify the signature validity. Thus, the signature used for acknowledgment and deacknowledgment of an edge follows the same format and can be used interchangeably. However, this design creates an issue, as described next.

1. Assume that Bob wants to acknowledge an edge. Thus, Bob calls the `acknowledgeEdge` function with his signature $Sig_1$.

2. A malicious user can always front-run Bob, take his signature ($Sig_1$) and sent to the `unacknowledgeEdge` function instead. When the `unacknowledgeEdge` function is executed with $Sig_1$, the signature will be marked as used by the code `_isUsed[keccak256(signature)] = true;`.

3. When Bob's `acknowledgeEdge` transaction gets executed, it will revert because his signature ($Sig_1$) has already been used.

The malicious users can keep repeating as the chain's gas fees on L2 are cheap.

The same trick can also be used to block someone from deacknowledge an edge.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L118

```solidity
File: TitlesGraph.sol
118:     function acknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
119:         external
120:         checkSignature(edgeId_, data_, signature_)
121:         returns (Edge memory edge)
122:     {
123:         return _setAcknowledged(edgeId_, data_, true);
124:     }
```

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L146

```solidity
File: TitlesGraph.sol
146:     function unacknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
147:         external
148:         checkSignature(edgeId_, data_, signature_)
149:         returns (Edge memory edge)
150:     {
151:         return _setAcknowledged(edgeId_, data_, false);
152:     }
```

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L40

```solidity
File: TitlesGraph.sol
39:     /// @notice Modified to check the signature for a proxied acknowledgment.
40:     modifier checkSignature(bytes32 edgeId, bytes calldata data, bytes calldata signature) {
41:         bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
42:         if (
43:             !edges[edgeId].to.creator.target.isValidSignatureNowCalldata(digest, signature)
44:                 || _isUsed[keccak256(signature)]
45:         ) {
46:             revert Unauthorized();
47:         }
48:         _;
49:         _isUsed[keccak256(signature)] = true;
50:     }
```

## Impact

Malicious users can block someone from acknowledging or deacknowledging an edge, affecting the sanctity of the data in the Graph.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L118

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L146

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L40

## Tool used

Manual Review

## Recommendation

```solidity
bytes32 public constant ACK_TYPEHASH = keccak256("Ack(bytes32 edgeId,bytes data)");
bytes32 public constant DEACK_TYPEHASH = keccak256("Deack(bytes32 edgeId,bytes data)");

bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
bytes32 digest = _hashTypedData(keccak256(abi.encode(DEACK_TYPEHASH, edgeId, data)));
```

Consider using two different hash types for acknowledging or deacknowledging within the signature and use a different modifier for checking the signature. This will prevent malicious users from taking the signature intended for `acknowledgeEdge` and submitting it to `unacknowledgeEdge`, and vice versa.

```solidity
modifier checkSignatureForAck(bytes32 edgeId, bytes calldata data, bytes calldata signature) {
    bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
    ..SNIP..
}

modifier checkSignatureForDeack(bytes32 edgeId, bytes calldata data, bytes calldata signature) {
    bytes32 digest = _hashTypedData(keccak256(abi.encode(DEACK_TYPEHASH, edgeId, data)));
    ..SNIP..
}
```

```diff
function acknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
    external
-   checkSignature(edgeId_, data_, signature_)
+   checkSignatureForAck(edgeId_, data_, signature_)
    returns (Edge memory edge)
{

function unacknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
    external
-   checkSignature(edgeId_, data_, signature_)
+		checkSignatureForDeack(edgeId_, data_, signature_)
    returns (Edge memory edge)
{
```

With this design, if a creator creates a signature intended for `acknowledgeEdge` function, and a malicious user front-runs and copies the signature and submits it to `acknowledgeEdge` function, no harm is done as the malicious user is simply executing the task on behalf of the creator. The edge will still be acknowledged at the end.



## Discussion

**pqseags**

Will investigate

# Issue M-7: Incorrect `supportsInterface` (EIP-165) 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/274 

## Found by 
CodeWasp, ZdravkoHr., cducrest-brainbot, jecikpo, xiaoming90
## Summary

The implementation of the `supportsInterface` (EIP-165) within `Edition` contract is incorrect, potentially leading to a loss of assets, as shown in the scenario below:

> A marketplace wants to determine if the `Edition` contract supports the [ERC-2981](https://eips.ethereum.org/EIPS/eip-2981) (NFT Royalty Standard) or has royalties by calling the `Edition.supportsInterface` function. Since the `Edition.supportsInterface` function returns false, the marketplace concluded that the Edition does not support ERC-2981 or does not have royalties. Thus, the marketplace will not collect royalties from the buyers and will not forward the royalties to the creators, leading to a loss of royalty fees for them.

## Vulnerability Detail

Per Line 36 below, the `Edition` contract supports the `ERC1155` and `ERC2981` interfaces. However, it was found that the `Edition.supportsInterface` function does not return `true` for `ERC1155` and `ERC2981` interfaces when being called, which indicates that it does not support the `ERC1155` and `ERC2981` interfaces.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L465

```solidity
File: Edition.sol
36: contract Edition is IEdition, ERC1155, ERC2981, Initializable, OwnableRoles {
..SNIP..
462:     /// @notice Check if the contract supports the given interface.
463:     /// @param interfaceId The interface ID to check.
464:     /// @return True if the contract supports the interface, false otherwise.
465:     function supportsInterface(bytes4 interfaceId)
466:         public
467:         view
468:         override(IEdition, ERC1155, ERC2981)
469:         returns (bool)
470:     {
471:         return super.supportsInterface(interfaceId);
472:     }
```

## Impact

Consider the following scenario that could potentially lead to a loss of assets due to incorrect `supportsInterface` function.

A marketplace wants to determine if the `Edition` contract supports the [ERC-2981](https://eips.ethereum.org/EIPS/eip-2981) (NFT Royalty Standard) or has royalties by calling the `Edition.supportsInterface` function. Since the `Edition.supportsInterface` function returns false, the marketplace concluded that the Edition does not support ERC-2981 or does not have royalties. Thus, the marketplace will not collect royalties from the buyers and will not forward the royalties to the creators, leading to a loss of royalty fees for them.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L465

## Tool used

Manual Review

## Recommendation

Consider updating the `supportsInterface` function to return true for the interfaces that the contract supports.

```diff
function supportsInterface(bytes4 interfaceId)
    public
    view
    override(IEdition, ERC1155, ERC2981)
    returns (bool)
{
-    return super.supportsInterface(interfaceId);
+    return
+        interfaceId == type(IERC1155).interfaceId ||
+        interfaceId == type(IERC2981).interfaceId ||
+        super.supportsInterface(interfaceId);
}
```



## Discussion

**pqseags**

Will address

# Issue M-8: Edition implementation not initialized on proxy 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/278 

## Found by 
mt030d, xiaoming90
## Summary

Edition implementation is not initialized on the proxy. As a result, the ability to create an edition, which is a core functionality of the protocol, is broken.

## Vulnerability Detail

The `TitlesCore` is the logic/implementation contract behind the ERC1967 proxy. The protocol adopts the UUPS proxy design.

It was observed that there is an issue at Line 37 below. When using a proxy contract, any direct initialization of state variables in the implementation contract (like `address public editionImplementation = address(new Edition());` directly in the field declaration) only affects the storage layout at the implementation level during deployment and does not affect the proxy’s state.

As a result, on the proxy itself, the `editionImplementation` will remain uninitialized, which is default to the `0x0000000000000000000000000000000000000000` (the zero address).

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L37

```solidity
File: TitlesCore.sol
32: contract TitlesCore is OwnableRoles, Initializable, UUPSUpgradeable, Receiver {
33:     using LibClone for address;
34:     using LibZip for bytes;
35:     using SafeTransferLib for address;
36: 
37:     address public editionImplementation = address(new Edition());
38:     FeeManager public feeManager;
39:     TitlesGraph public graph;
40: 
41:     /// @notice Initializes the protocol.
42:     /// @param feeReceiver_ The address to receive fees.
43:     /// @param splitFactory_ The address of the split factory.
44:     function initialize(address feeReceiver_, address splitFactory_) external initializer {
45:         _initializeOwner(msg.sender);
46: 
47:         feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
48:         graph = new TitlesGraph(address(this), msg.sender);
49:     }
```

When someone creates a new Edition, the Edition contract will be cloned from the `editionImplementation` as shown at Line 79 below. Since the `editionImplementation` is not initialized, the clone will fail and revert. As a result, users will not be able to create an edition.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L79

```solidity
File: TitlesCore.sol
68:     /// @notice Creates an {Edition} with the given payload.
69:     /// @param payload_ The compressed payload for creating the {Edition}. See {EditionPayload}.
70:     /// @param referrer_ The address of the referrer.
71:     /// @return edition The new {Edition}.
72:     function createEdition(bytes calldata payload_, address referrer_)
73:         external
74:         payable
75:         returns (Edition edition)
76:     {
77:         EditionPayload memory payload = abi.decode(payload_.cdDecompress(), (EditionPayload));
78: 
79:         edition = Edition(editionImplementation.clone());
```

## Impact

Creating an edition is a core functionality of the protocol and this function is broken. 

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L37

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L79

## Tool used

Manual Review

## Recommendation

Consider the following changes to ensure that the `editionImplementation` is initialized properly on the proxy.

```diff
- address public editionImplementation = address(new Edition());
+ address public editionImplementation;

function initialize(address feeReceiver_, address splitFactory_) external initializer {
    _initializeOwner(msg.sender);

    feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
    graph = new TitlesGraph(address(this), msg.sender);
+   editionImplementation = address(new Edition());
}
```



## Discussion

**pqseags**

Will fix

# Issue M-9: Broken batch minting feature 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/280 

## Found by 
Varun\_05, ast3ros, kn0t, trachev, valentin2304, xiaoming90
## Summary

The core minting feature of the protocol is broken due to the mishandling of `msg.value` within the for-loop.

## Vulnerability Detail

Assume that the total fee for each token is 0.001 ETH, and Bob wants to mint four tokens. The total fee will be 0.004 ETH, so he will send 0.004 ETH when calling the above `mintBatch` function.

An important point to note is that the `msg.value` will always remain at 0.004 ETH throughout the entire execution of the `mintBatch` function. The `msg.value` will not automatically be reduced regardless of how many ETH has been transferred out or "spent".

In the first for-loop, the `msg.value` will be 0.004 ETH, and all 0.004 ETH will be routed to the fee manager and subsequently routed to the fee recipient address/0xSplit wallet.

In the second for-loop, since all the ETH (0.004 ETH) was sent to the fee manager earlier, the amount of ETH left on the Edition contract is zero. When the second for-loop attempts to send `msg.value` (0.004 ETH) to the fee manager again, it will revert due to insufficient ETH, and the transaction will fail and revert. Thus, this batch minting feature is broken.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L277

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
288:                 this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy
289:             );
290: 
291:             _checkTime(work.opensAt, work.closesAt);
292:             _updateSupply(work, amounts_[i]);
293:         }
294: 
295:         _batchMint(to_, tokenIds_, amounts_, data_);
296:         _refundExcess();
297:     }
```

## Impact

Breaks core contract functionality. The batch minting feature, a core feature of the protocol, is broken.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L277

## Tool used

Manual Review

## Recommendation

For each loop, consider only forwarding/transferring the minting fee for the current token ID instead of the entire ETH (`msg.value`).

```diff
function mintBatch(
    address to_,
    uint256[] calldata tokenIds_,
    uint256[] calldata amounts_,
    bytes calldata data_
) external payable {
    for (uint256 i = 0; i < tokenIds_.length; i++) {
        Work storage work = works[tokenIds_[i]];
+				uint256 mintFee = FEE_MANAGER.getMintFee(work.strategy, amounts_[i])

        // wake-disable-next-line reentrancy
+        FEE_MANAGER.collectMintFee{value: mintFee}(
-        FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy 
        );
```



## Discussion

**pqseags**

I don't think the duplicates on this issue are actually duplicates of this issue. @Hash01011122 

# Issue M-10: EIP 712 is not implemented for the `Edition` contract. 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/282 

The protocol has acknowledged this issue.

## Found by 
xiaoming90
## Summary

EIP 712 is not implemented for the `Edition` contract.

## Vulnerability Detail

The Contest's README stated that EIP 712 must be strictly implemented for the `Edition` contract. Thus, in the context of this audit contest, any non-compliance is considered a valid Medium issue. Note that the Contest's README supersedes Sherlock's judging rules per Sherlock's Hierarchy of truth.

> Is the codebase expected to comply with any EIPs? Can there be/are there any deviations from the specification?
>
> **strict implementation of EIPs**
> 1271 (Graph), **712 (Graph, Edition)**, 2981 (Edition), 1155 (Edition)

However, when reviewing the `Edition` contract, it was observed that EIP 712 is not implemented for the `Edition` contract. Thus, breaking the above requirement.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L36

```solidity
File: Edition.sol
34: /// @title Edition
35: /// @notice An ERC1155 contract representing a collection of related works. Each work is represented by a token ID.
36: contract Edition is IEdition, ERC1155, ERC2981, Initializable, OwnableRoles {
37:     using SafeTransferLib for address;
..SNIP..
```

## Impact

EIP 712 is not implemented for the `Edition` contract.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L36

## Tool used

Manual Review

## Recommendation

Consider implementing EIP 712 for the `Edition` contract.



## Discussion

**pqseags**

This seems valid based on the Readme

**Hash01011122**

Seems borderline low/medium 

# Issue M-11: New creators unable to update the royalty target and the fee route for their works 

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/283 

## Found by 
0x73696d616f, BengalCatBalu, BiasedMerc, CodeWasp, KupiaSec, Varun\_05, jennifer37, mt030d, sammy, xiaoming90, y4y
## Summary

The new creators are unable to update the royalty target and the fee route for their works. As a result, it could lead to a loss of assets for the new creator due to the following:

- The work's royalty target still points to the previous creator, so the royalty fee is routed to the previous creator instead of the current creator.
- The work's fee is still routed to the recipients, which included the previous creator instead of the current creator.

## Vulnerability Detail

The creator can call the `transferWork` to transfer the work to another creator. However, it was observed that after the transfer, there is still much information about the work pointing to the previous creator instead of the new creator.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L412

```solidity
File: Edition.sol
412:     function transferWork(address to_, uint256 tokenId_) external {
413:         Work storage work = works[tokenId_];
414:         if (msg.sender != work.creator) revert Unauthorized();
415: 
416:         // Transfer the work to the new creator
417:         work.creator = to_;
418: 
419:         emit WorkTransferred(address(this), tokenId_, to_);
420:     }
```

Following are some instances of the issues:

- The work's royalty target still points to the previous creator, so the royalty fee is routed to the previous creator instead of the current creator.
- The work's fee is still routed to the recipients, which included the previous creator instead of the current creator.

To aggravate the issues, creators cannot call the `Edition.setRoyaltyTarget` and `FeeManager.createRoute` as these functions can only executed by EDITION_MANAGER_ROLE and [Owner, Admin], respectively.

Specifically for the `Edition.setRoyaltyTarget` function that can only be executed by EDITION_MANAGER_ROLE, which is restricted and not fully trusted in the context of this audit. The new creator could have purchased the work from the previous creator, but only to find out that the malicious edition manager decided not to update the royalty target to point to the new creator's address for certain reasons, leading to the royalty fee continuing to be routed to the previous creator. In this case, it negatively impacts the new creator as it leads to a loss of royalty fee for the new creator.

> [!NOTE]
>
> The following is an extract from the contest's README stating that the EDITION_MANAGER_ROLE is restricted. This means that any issue related to EDITION_MANAGER_ROLE that could affect TITLES protocol/users negatively will be considered valid in this audit contest.
>
> > EDITION_MANAGER_ROLE (Restricted) =>

## Impact

Loss of assets for the new creator due to the following:

- The work's royalty target still points to the previous creator, so the royalty fee is routed to the previous creator instead of the current creator.
- The work's fee is still routed to the recipients, which included the previous creator instead of the current creator.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L412

## Tool used

Manual Review

## Recommendation

Consider allowing the creators themselves to have the right to update the royalty target and the fee route for their works.



## Discussion

**pqseags**

While this is unintuitive, we do not intend to update the fee splits when ownership of a work is changed after publishing

