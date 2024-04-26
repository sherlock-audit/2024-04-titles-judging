Winning Scarlet Yeti

medium

# Broken batch minting feature

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