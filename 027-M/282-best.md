Winning Scarlet Yeti

medium

# EIP 712 is not implemented for the `Edition` contract.

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