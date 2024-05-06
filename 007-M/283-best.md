Winning Scarlet Yeti

medium

# New creators unable to update the royalty target and the fee route for their works

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