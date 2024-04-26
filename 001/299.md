Daring Carob Gorilla

high

# `_refundExcess` leads to funds lost

## Summary
`_refundExcess` does not work leading to funds lost.
## Vulnerability Detail
If a user send more ETH than required, `_refundExcess` should refund the excess amount:
```javascript
    function mint(
        address to_,
        uint256 tokenId_,
        uint256 amount_,
        address referrer_,
        bytes calldata data_
    ) external payable override {
        FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
        );
        _issue(to_, tokenId_, amount_, data_);
->      _refundExcess();
    }
    
	function _refundExcess() internal {
        if (msg.value > 0 && address(this).balance > 0) {
            msg.sender.safeTransferETH(address(this).balance);
        }
    }
```
Problem is that `address(this).balance` will be `0`, resulting in `msg.sender.safeTransferETH(address(this).balance)` not executing.

Furthermore, any person can monitor the balance of the Edition contract and just mint a (free) Edition with more than `1 wei` to suffice this check:
- `if (msg.value > 0 && address(this).balance > 0)`
and empty the contract.
## Proof of Concept
Put this in `Edition.t.sol`:
```javascript
    function test_poc() public {
        // mintFee
        uint256 mintFee = 0.01 ether;
        // Alice gets 1 ether
        address alice = makeAddr("alice");
        vm.deal(alice, 1 ether);

        // Logs
        console.log("Alice balance:");
        console.log(alice.balance);
        
        // Mint as Alice with 1 ether, she should get a refund
        vm.prank(alice);
        edition.mint{value: 1 ether}(address(1), 1, 1, address(0), new bytes(0));

        // Balance will be 0 even though she should've been refunded
        vm.assertEq(alice.balance, 0);

        // Logs
        console.log("Alice balance:");
        console.log(alice.balance);
    }
```

Run this, this results in:
```javascript
[PASS] test_poc() (gas: 181054)
Logs:
  Alice balance:
  1000000000000000000
  Alice balance:
  0
```
Alice did not get refunded leading  to Alices' funds being lost.
## Impact
Funds lost due to refunds not working correctly.
## Code Snippet
[Edition.sol#L512-L516](https://github.com/sherlock-audit/2024-04-titles/blob/c9d16782a7d3c15c7a759f22c9e0552d5e777ed7/wallflower-contract-v2/src/editions/Edition.sol#L512-L516)
## Tool used
Manual Review
## Recommendation
Modify the `_refundExcess()` function.