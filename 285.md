Winning Scarlet Yeti

medium

# Malicious EDITION_MANAGER_ROLE can front-run victims to increase royalty

## Summary

Malicious EDITION_MANAGER_ROLE can front-run victims to increase royalty, leading to a loss of assets for the victim.

## Vulnerability Detail

The following is an extract from the contest's README stating that the EDITION_MANAGER_ROLE is restricted. This means that any issue related to EDITION_MANAGER_ROLE that could affect TITLES protocol/users negatively will be considered valid in this audit contest.

> EDITION_MANAGER_ROLE (Restricted) =>
> On an Edition, this role can:
>
> Set the Edition's ERC2981 royalty receiver (`setRoyaltyTarget`). This is the only way to change the royalty receiver for the Edition.

> [!NOTE]
>
> **About ERC2981**
>
> ERC2981 known as "NFT Royalty Standard." It introduces a standardized way to handle royalty payments for non-fungible tokens (NFTs) on the Ethereum blockchain, providing a mechanism to ensure creators receive a share of the proceeds when their NFTs are resold

Assume that the current royalty for work/collection X ($Collection_X$) is 5%. When secondary marketplaces resell the TITLES NFT, they will retrieve the NFT's royalty information by calling the `Edition.royaltyInfo` function. The royalty payments are calculated, collected, and transferred automatically to the creator's wallet whenever their NFT is resold.

The issue is that the EDITION_MANAGER_ROLE is restricted in the context of this audit and is considered not fully trusted. 

Assume Bob submits a transaction to purchase an NFT from $Collection_X$​ to the mempool. A malicious editor manager could front-run the purchase transaction and attempt to increase the royalty to be increased from 5% to 95%, resulting in more royalty fees being collected from Bob, and routed to the creators. This leads to a loss of assets for Bob.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L389

```solidity
File: Edition.sol
386:     /// @notice Set the ERC2981 royalty target for the given work.
387:     /// @param tokenId The ID of the work.
388:     /// @param target The address to receive royalties.
389:     function setRoyaltyTarget(uint256 tokenId, address target)
390:         external
391:         onlyRoles(EDITION_MANAGER_ROLE)
392:     {
393:         _setTokenRoyalty(tokenId, target, works[tokenId].strategy.royaltyBps);
394:     }
```

## Impact

Loss of assets for the victim, as shown in the above scenario

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L389

## Tool used

Manual Review

## Recommendation

Ensure that only trusted users can update the royalty information, as this is a sensitive value that could affect the fee being charged when TITLES NFT is resold in the secondary market.