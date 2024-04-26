Winning Scarlet Yeti

high

# Excess ETH of the victim can be stolen by malicious external parties due to re-entrancy attack

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