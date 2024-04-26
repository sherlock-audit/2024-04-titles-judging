Winning Scarlet Yeti

medium

# Number of attributions is not restricted

## Summary

The number of of attributions is not restricted, leading to the following issues:

- None of the attributions will receive their share of the fee due to rounding down to zero.
- Minting of tokens will be broken as a revert caused by Out-of-Gas will occur when the code attempts to route the minting fee to the fee recipients.

## Vulnerability Detail

It was observed that the number of attributions that can be configured when publishing/creating a new work/collection is not restricted. As a result, it will lead to the following issues:

#### Issue 1

Assume that `revshareBps` (royalty revenue share) is set to 50 (0.5%). If the number of `attributionShares` is 5001 or more, the amount of revenue share received by the attribution (`attributionRevShare`) will round down to zero, and none of the attributions will receive any royalty `revshare` allocated to them.

```solidity
attributionRevShare = revshareBps * 100 / attributionShares
attributionRevShare = 50 * 100 / 5001 = 0
```

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L476

```solidity
File: FeeManager.sol
476:     function _buildSharesAndTargets(
477:         Target memory creator,
478:         Target[] memory attributions,
479:         uint32 revshareBps
480:     ) internal pure returns (address[] memory targets, uint256[] memory shares) {
481:         uint32 attributionShares = uint32(attributions.length);
482:         uint32 attributionRevShare = revshareBps * 100 / attributionShares;
483:         uint32 creatorShare = 1e6 - (attributionRevShare * attributionShares);

```

#### Issue 2

If the number of attributions is too large when the mint fee is collected from the users, and the fee is routed to a large number of attributions, an out-of-gas (OOG) error will occur. As a result, the entire minting process will revert, and the minting of Token ID cannot proceed further.

If the number of attributions is large, the number of recipients of the newly deployed 0xSplit wallet will also be large per Line 146 below. The fact that a large number of recipients can lead to a revert in 0xSplit wallet or locked fund is a known issue that has been flagged out during [0xSplit audit report](https://github.com/zobront/audits/blob/main/reports/splits-v2.md#m-01-splits-with-too-many-recipients-can-lead-to-locked-funds).

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L146

```solidity
File: FeeManager.sol
125:     function createRoute(
126:         IEdition edition_,
127:         uint256 tokenId_,
128:         Target[] calldata attributions_,
129:         address referrer_
130:     ) external onlyOwnerOrRoles(ADMIN_ROLE) returns (Target memory receiver) {
..SNIP..
136:         } else {
137:             // Distribute the fee among the creator and attributions
138:             (address[] memory targets, uint256[] memory revshares) = _buildSharesAndTargets(
139:                 creator, attributions_, edition_.feeStrategy(tokenId_).revshareBps
140:             );
141: 
142:             // Create the split. The protocol retains "ownership" to enable future use cases.
143:             receiver = Target({
144:                 target: splitFactory.createSplit(
145:                     SplitV2Lib.Split({
146:                         recipients: targets,
147:                         allocations: revshares,
148:                         totalAllocation: 1e6,
149:                         distributionIncentive: 0
150:                     }),
151:                     address(this),
152:                     creator.target
153:                     ),
154:                 chainId: creator.chainId
155:             });
156:         }
```

## Impact

Issue 1 - None of the attributions will receive their share of the fee due to rounding down to zero.

Issue 2 - Minting of tokens will be broken as a revert caused by Out-of-Gas will occur when the code attempts to route the minting fee to the fee recipients.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L476

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L146

## Tool used

Manual Review

## Recommendation

Consider restricting the number of attributions to a reasonable value, such as ~250 recipients/attributions.

```diff
function createRoute(
    IEdition edition_,
    uint256 tokenId_,
    Target[] calldata attributions_,
    address referrer_
) external onlyOwnerOrRoles(ADMIN_ROLE) returns (Target memory receiver) {
		..SNIP..
    } else {
        // Distribute the fee among the creator and attributions
        (address[] memory targets, uint256[] memory revshares) = _buildSharesAndTargets(
            creator, attributions_, edition_.feeStrategy(tokenId_).revshareBps
        );
+       if (targets.length > 250) revert InvalidAttributions_TooManyRecipients();
```