Winning Scarlet Yeti

high

# Excess ETH will be stuck in the Fee Manager contract and not swept back to the users

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