Perfect Lipstick Anteater

high

# Mint Functions in `Edition` Contract Fail to Refund Excess ETH Sent by Users

## Summary
In the `Edition` contract, when users mint tokens through any of the mint functions (`mint`, `mintWithComment` and two `mintBatch`), excess ETH is not refunded. Although the `_refundExcess` function is called at the end to manage this refund, the implementation within the mint functions incorrectly handles the logic, leading to the issue.

## Vulnerability Detail
When a user initiates a mint function, all of the `msg.value` is sent to the `feeManager` contract to collect fees. However, since the `feeManager` does not facilitate the return of excess ETH, any surplus remains within the `feeManager` contract. Consequently, the balance of the `Edition` contract stays at zero, preventing `_refundExcess()` from refunding any ETH to the user.

Consider the `mint` function as an example (the issue persists similarly across other mint functions):
```javascript
        function mint(
        address to_,
        uint256 tokenId_,
        uint256 amount_,
        address referrer_,
        bytes calldata data_
    ) external payable override {
        // wake-disable-next-line reentrancy
@>      FEE_MANAGER.collectMintFee{value: msg.value}(
            this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
        );

        _issue(to_, tokenId_, amount_, data_);

        // @audit Excess ETH remains stuck in feeManager
        _refundExcess();
    }
```

## Impact
Users who mint tokens using the `Edition` contract may lose the excess funds they send. This can result in financial losses and dissatisfaction among users.

## Code Snippet


### Here is a test for PoC:
Insert the following test in the [`Edition.t.sol`](https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/test/editions/Edition.t.sol#L175) file 

```javascript
    function test_mintExtraEth() public {
        uint mintFee = edition.mintFee(1);
        
        // Capturing the balance before and after the mint to ensure the mint fee is paid and excess eth is returned
        uint balanceBefore = address(this).balance;
        
        // Minting with extra 10 eth
        edition.mint{value: mintFee + 10 ether}(address(1), 1, 1, address(0), new bytes(0));
        uint balanceAfter = address(this).balance;
        
        // Ensuring the excess eth is returned. but this assertion fails because excess ETH is left in the feeManager
        assertEq(balanceBefore - mintFee, balanceAfter);
        
        assertEq(edition.totalSupply(1), 1);
        assertEq(address(1).balance, 0.01 ether);
        assertEq(address(0xc0ffee).balance, 0.0006 ether);
    }

    // Enables the contract to receive excess ETH, which is necessary after fixing the minting issue.
    receive() external payable {}
```

## Tool used

Manual Review

## Recommendation
Modify the mint functions to only send the required fee to the `feeManager` and refund any excess ETH to the user. This can be implemented by adjusting the mint functions as shown in the diffs below:

```diff
    function mint(
        address to_,
        uint256 tokenId_,
        uint256 amount_,
        address referrer_,
        bytes calldata data_
    ) external payable override {
        // wake-disable-next-line reentrancy
-       FEE_MANAGER.collectMintFee{value: msg.value}(
+       FEE_MANAGER.collectMintFee{value: mintFee(tokenId_)}(
            this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
        );

        _issue(to_, tokenId_, amount_, data_);

        _refundExcess();
    }
```

```diff
   function mintWithComment(
        address to_,
        uint256 tokenId_,
        uint256 amount_,
        address referrer_,
        bytes calldata data_,
        string calldata comment_
    ) external payable {
        Strategy memory strategy = works[tokenId_].strategy;
        // wake-disable-next-line reentrancy
-       FEE_MANAGER.collectMintFee{value: msg.value}(
+       FEE_MANAGER.collectMintFee{value: mintFee(tokenId_)}(
            this, tokenId_, amount_, msg.sender, referrer_, strategy
        );

        _issue(to_, tokenId_, amount_, data_);
        _refundExcess();

        emit Comment(address(this), tokenId_, to_, comment_);
    }
```

```diff
function mintBatch(
        address to_,
        uint256[] calldata tokenIds_,
        uint256[] calldata amounts_,
        bytes calldata data_
    ) external payable {
        for (uint256 i = 0; i < tokenIds_.length; i++) {
            Work storage work = works[tokenIds_[i]];

            // wake-disable-next-line reentrancy
-          FEE_MANAGER.collectMintFee{value: msg.value}(
+          FEE_MANAGER.collectMintFee{value: mintFee(tokenIds_[i])}(
                this, tokenIds_[i], amounts_[i], msg.sender, address(0), work.strategy
            );

            _checkTime(work.opensAt, work.closesAt);
            _updateSupply(work, amounts_[i]);
        }

        _batchMint(to_, tokenIds_, amounts_, data_);
        _refundExcess();
    }
```

```diff
    function mintBatch(
        address[] calldata receivers_,
        uint256 tokenId_,
        uint256 amount_,
        bytes calldata data_
    ) external payable {
        // wake-disable-next-line reentrancy
-       FEE_MANAGER.collectMintFee{value: msg.value}(
+       FEE_MANAGER.collectMintFee{value: mintFee(tokenId_)}(
            this, tokenId_, amount_, msg.sender, address(0), works[tokenId_].strategy
        );

        for (uint256 i = 0; i < receivers_.length; i++) {
            _issue(receivers_[i], tokenId_, amount_, data_);
        }

        _refundExcess();
    }
```