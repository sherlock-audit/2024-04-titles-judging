Winning Scarlet Yeti

medium

# Incorrect `supportsInterface` (EIP-165)

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