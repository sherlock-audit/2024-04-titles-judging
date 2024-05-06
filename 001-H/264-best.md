Winning Scarlet Yeti

high

# Users can exploit the batch minting feature to avoid paying minting fees for tokens

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