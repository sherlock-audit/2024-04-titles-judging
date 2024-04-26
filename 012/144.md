Fun Banana Chameleon

medium

# ERC2981 royalties discrepancy with strategy

## Summary

In `Edition.sol`, functions that set the value of `works[tokenId].strategy` which includes `works[tokenId].strategy.royaltyBps` do not set ERC2981's internal token royalty value.

## Vulnerability Detail

The function `setFeeStrategy()` sets the public mapping value `works[tokenId_].strategy` which may update the `roylatyBps` value but does not call `_setTokenRoyalty(...)`:

```solidity
    function setFeeStrategy(uint256 tokenId_, Strategy calldata strategy_) external {
        if (msg.sender != works[tokenId_].creator) revert Unauthorized();
        works[tokenId_].strategy = FEE_MANAGER.validateStrategy(strategy_);  // @audit does not set royalties
    }
```

Similarly, the `publish()` function to create a new work sets the strategy but does not call `_setTokenRoyalty()`:

```solidity
    function publish(
        address creator_,
        uint256 maxSupply_,
        uint64 opensAt_,
        uint64 closesAt_,
        Node[] calldata attributions_,
        Strategy calldata strategy_,
        Metadata calldata metadata_
    ) external override onlyRoles(EDITION_MANAGER_ROLE) returns (uint256 tokenId) {
        tokenId = ++totalWorks;

        _metadata[tokenId] = metadata_;
        works[tokenId] = Work({
            creator: creator_,
            totalSupply: 0,
            maxSupply: maxSupply_,
            opensAt: opensAt_,
            closesAt: closesAt_,
            strategy: FEE_MANAGER.validateStrategy(strategy_)
        });

        Node memory _node = node(tokenId);
        for (uint256 i = 0; i < attributions_.length; i++) {
            // wake-disable-next-line reentrancy, unchecked-return-value
            GRAPH.createEdge(_node, attributions_[i], attributions_[i].data);
        }

        emit Published(address(this), tokenId);
    } 
```

This latter case may be less of a problem since `TitlesCore` calls `edition_.setRoyaltyTarget()` right after publishing a new work. However it remains a problem if publishers are expected to interact directly with `Edition` and not only through `TitlesCore`

## Impact

The value returned by `works[tokenId].strategy.royaltyBps` and `ERC2981.royaltyInfo(tokenId, salePrice)` will not be coherent. Users may expect to set a certain royalty bps while the value is not updated. Core values used for royalty payments may become incorrect after updates.

## Code Snippet

https://github.com/vectorized/solady/blob/main/src/tokens/ERC2981.sol

https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/editions/Edition.sol#L368-L371

https://github.com/sherlock-audit/2024-04-titles/blob/d7f60952df22da00b772db5d3a8272a988546089/wallflower-contract-v2/src/editions/Edition.sol#L121

## Tool used

Manual Review

## Recommendation

Call `_setTokenRoyalty(tokenId, FeeManager.feeReceiver(address(this), tokenId), works[tokenId].strategy.royaltyBps);` at the end of `setFeeStrategy()`. 
