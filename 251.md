Cheery Honeysuckle Tuna

high

# `TitlesCore._publish()` can be DOSd by consuming the user's allowance by frontrunning `FeeManager.collectCreationFee()`

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