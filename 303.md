Daring Carob Gorilla

high

# Creating editions can fail and creating routes can fail due to un-ordering of targets

## Summary
Due to an un-ordered array, `TitlesCore.createEdition` and `FeeManager.createRoute()` can fail.
## Vulnerability Detail
During the `TitlesCore.createEdition()` execution flow, inside `FeeManager.createRoute()`, `splitFactory.createSplit()` is called:
```javascript
    function createRoute(
        IEdition edition_,
        uint256 tokenId_,
        Target[] calldata attributions_,
        address referrer_
    ) external onlyOwnerOrRoles(ADMIN_ROLE) returns (Target memory receiver) {
        // ommitted code
         receiver = Target({
->              target: splitFactory.createSplit(
                    SplitV2Lib.Split({
                        recipients: targets,
                        allocations: revshares,
                        totalAllocation: 1e6,
                        distributionIncentive: 0
                    }),
                    address(this),
                    creator.target
                    ),
		// ommitted code
    }

```

If we take a look at `splitFactory.createSplit()`, we'll see that the passed accounts are required to be ordered. 
See this comment:
[SplitMain.sol#L250](https://github.com/0xSplits/splits-contracts/blob/c7b741926ec9746182d0d1e2c4c2046102e5d337/contracts/SplitMain.sol#L250)
```md
*  @param accounts Ordered, unique list of addresses with ownership in the split
```

This check happens in the modifier `validSplit()`:
[SplitMain.sol#L184-L217](https://github.com/0xSplits/splits-contracts/blob/c7b741926ec9746182d0d1e2c4c2046102e5d337/contracts/SplitMain.sol#L184-L217)
```javascript
  modifier validSplit(
    address[] memory accounts,
    uint32[] memory percentAllocations,
    uint32 distributorFee
  ) {
    if (accounts.length < 2)
      revert InvalidSplit__TooFewAccounts(accounts.length);
    if (accounts.length != percentAllocations.length)
      revert InvalidSplit__AccountsAndAllocationsMismatch(
        accounts.length,
        percentAllocations.length
      );
    // _getSum should overflow if any percentAllocation[i] < 0
    if (_getSum(percentAllocations) != PERCENTAGE_SCALE)
      revert InvalidSplit__InvalidAllocationsSum(_getSum(percentAllocations));
    unchecked {
      // overflow should be impossible in for-loop index
      // cache accounts length to save gas
      uint256 loopLength = accounts.length - 1;
      for (uint256 i = 0; i < loopLength; ++i) {
        // overflow should be impossible in array access math
->      if (accounts[i] >= accounts[i + 1])
          revert InvalidSplit__AccountsOutOfOrder(i);
        if (percentAllocations[i] == uint32(0))
          revert InvalidSplit__AllocationMustBePositive(i);
      }
// overflow should be impossible in array access math with validated equal array lengths
      if (percentAllocations[loopLength] == uint32(0))
        revert InvalidSplit__AllocationMustBePositive(loopLength);
    }
    if (distributorFee > MAX_DISTRIBUTOR_FEE)
      revert InvalidSplit__InvalidDistributorFee(distributorFee);
    _;
  }
```
This means that the `TitlesCore.createEdition()` function and the `FeeManager.createRoute()` function will end up reverting with the error: `InvalidSplit__AccountsOutOfOrder()` if the accounts are not ordered.

The accounts passed to the `splitFactory.createSplit()` function get built inside the `FeeManager._buildSharesAndTargets()` function:
```javascript
	function createRoute(
        IEdition edition_,
        uint256 tokenId_,
        Target[] calldata attributions_,
        address referrer_
    ) external onlyOwnerOrRoles(ADMIN_ROLE) returns (Target memory receiver) {
    // ommitted code
            (address[] memory targets, uint256[] memory revshares) = _buildSharesAndTargets(
                creator, attributions_, edition_.feeStrategy(tokenId_).revshareBps
            );
	// ommitted code
	}
        
	function _buildSharesAndTargets(
        Target memory creator,
        Target[] memory attributions,
        uint32 revshareBps
    ) internal pure returns (address[] memory targets, uint256[] memory shares) {
        uint32 attributionShares = uint32(attributions.length);
        uint32 attributionRevShare = revshareBps * 100 / attributionShares;
        uint32 creatorShare = 1e6 - (attributionRevShare * attributionShares);
        targets = new address[](attributionShares + 1);
        shares = new uint256[](attributionShares + 1);
        targets[0] = creator.target;
        shares[0] = creatorShare;
        for (uint8 i = 0; i < attributionShares; i++) {
            targets[i + 1] = attributions[i].target;
            shares[i + 1] = attributionRevShare;
        }
    }

```
As we can see, the first target will always be set to `creator.target`:
```javascript
		targets[0] = creator.target;
```

This means that, `splitFactory.createSplit()` will always fail if `target[1..n] < creator.target`, meaning people will not be able to `TitlesCore.createEdition()` depending on the `attributions` supplied and it will not be possible to `FeeManager.createRoute()` depending on the `attributions` supplied. 

This is especially a problem because the first address of the `target[]` is always set to `creator.target` , which means it can not be swapped another index to keep the array in order.
## Proof of Concept

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";

interface ISplitFactory {
  function createSplit(
    address[] calldata accounts,
    uint32[] calldata percentAllocations,
    uint32 distributorFee,
    address controller
  )
    external
    returns (address split);
}

contract TitlesE2E is Test {
  // https://etherscan.io/address/0x2ed6c4B5dA6378c7897AC67Ba9e43102Feb694EE#code
  address split_factory_address = 0x2ed6c4B5dA6378c7897AC67Ba9e43102Feb694EE;
  ISplitFactory splitFactory = ISplitFactory(split_factory_address);

  function setUp() public {
    // Create the fork
    // Input your RPC here
    vm.createSelectFork("inputRPCHere");
  }

  function test_poc() public {
    // Sanity check
    // We took a legit transaction that was made on the ETH mainnet - this will pass
    // https://etherscan.io/tx/0x3ed250d745f21cd8d2ee454fe4a97eb661cda62216bd33bd056c79242e6eaec3
    address(splitFactory).call(hex"7601f782000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000002c00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000c01b3424b81da02a3ab626a44cfe67c2eb15d9da0000000000000000000000000000000000000000000000000000000000000011000000000000000000000000077aac78a05f5a116fde61cd4ef3081c175315c2000000000000000000000000133890338e33141727b2252cc92f3081ca2120800000000000000000000000001bcaff43a8607b0416d9e71586498a16c3486f0300000000000000000000000029d489e1c200740d91fce85614a0a13376eda87a00000000000000000000000046761f1d443531afc94080e9f93d7aad7c75ae87000000000000000000000000644e6abb3f9e8754bd9e07159f4ee7d5e3c5c4ac0000000000000000000000006d62d8e11b342d4d1f3054dd4c9d9540fea76b3a00000000000000000000000087c926b07162be81ed2af05e3307a7d4a3f0ade10000000000000000000000008ed3e58fcf1f2169f7b39fc1aa8d753719ec53a800000000000000000000000094d94ef8422418fc5e8aba0db5900a3454d8029e000000000000000000000000a05a94773154de8c0eed5fdf7d1889efea02dc9a000000000000000000000000a7ecbb281fb3fe21943928282cb4452279b4a65c000000000000000000000000b5c1a33bbc95ce8541af2e9c407f3989f9dabdd0000000000000000000000000b74f97da97609ed9a592a12a4641ebc5cd602406000000000000000000000000c01b3424b81da02a3ab626a44cfe67c2eb15d9da000000000000000000000000db9ae004be5b96125685756c75b177189d5b47a6000000000000000000000000e3525546721893aa484dda2b7139ba71d751be5e000000000000000000000000000000000000000000000000000000000000001100000000000000000000000000000000000000000000000000000000000053020000000000000000000000000000000000000000000000000000000000019f0a0000000000000000000000000000000000000000000000000000000000005302000000000000000000000000000000000000000000000000000000000000a6040000000000000000000000000000000000000000000000000000000000005302000000000000000000000000000000000000000000000000000000000001afa4000000000000000000000000000000000000000000000000000000000000530200000000000000000000000000000000000000000000000000000000000053020000000000000000000000000000000000000000000000000000000000005302000000000000000000000000000000000000000000000000000000000000a604000000000000000000000000000000000000000000000000000000000000530200000000000000000000000000000000000000000000000000000000000056ea000000000000000000000000000000000000000000000000000000000004cadb000000000000000000000000000000000000000000000000000000000000a604000000000000000000000000000000000000000000000000000000000001f4af00000000000000000000000000000000000000000000000000000000000053020000000000000000000000000000000000000000000000000000000000005302");

  // However, if we change the first address from:
  // 0x077AAC78a05F5a116Fde61Cd4eF3081c175315c2
  // To:
  // 0x177AAC78a05F5a116Fde61Cd4eF3081c175315c2
  // It will revert because:
  // 0x177AAC78a05F5a116Fde61Cd4eF3081c175315c2 > 0x133890338E33141727B2252Cc92F3081Ca212080

    vm.expectRevert();
    address(splitFactory).call(hex"7601f782000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000002c00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000c01b3424b81da02a3ab626a44cfe67c2eb15d9da0000000000000000000000000000000000000000000000000000000000000011000000000000000000000000177aac78a05f5a116fde61cd4ef3081c175315c2000000000000000000000000133890338e33141727b2252cc92f3081ca2120800000000000000000000000001bcaff43a8607b0416d9e71586498a16c3486f0300000000000000000000000029d489e1c200740d91fce85614a0a13376eda87a00000000000000000000000046761f1d443531afc94080e9f93d7aad7c75ae87000000000000000000000000644e6abb3f9e8754bd9e07159f4ee7d5e3c5c4ac0000000000000000000000006d62d8e11b342d4d1f3054dd4c9d9540fea76b3a00000000000000000000000087c926b07162be81ed2af05e3307a7d4a3f0ade10000000000000000000000008ed3e58fcf1f2169f7b39fc1aa8d753719ec53a800000000000000000000000094d94ef8422418fc5e8aba0db5900a3454d8029e000000000000000000000000a05a94773154de8c0eed5fdf7d1889efea02dc9a000000000000000000000000a7ecbb281fb3fe21943928282cb4452279b4a65c000000000000000000000000b5c1a33bbc95ce8541af2e9c407f3989f9dabdd0000000000000000000000000b74f97da97609ed9a592a12a4641ebc5cd602406000000000000000000000000c01b3424b81da02a3ab626a44cfe67c2eb15d9da000000000000000000000000db9ae004be5b96125685756c75b177189d5b47a6000000000000000000000000e3525546721893aa484dda2b7139ba71d751be5e000000000000000000000000000000000000000000000000000000000000001100000000000000000000000000000000000000000000000000000000000053020000000000000000000000000000000000000000000000000000000000019f0a0000000000000000000000000000000000000000000000000000000000005302000000000000000000000000000000000000000000000000000000000000a6040000000000000000000000000000000000000000000000000000000000005302000000000000000000000000000000000000000000000000000000000001afa4000000000000000000000000000000000000000000000000000000000000530200000000000000000000000000000000000000000000000000000000000053020000000000000000000000000000000000000000000000000000000000005302000000000000000000000000000000000000000000000000000000000000a604000000000000000000000000000000000000000000000000000000000000530200000000000000000000000000000000000000000000000000000000000056ea000000000000000000000000000000000000000000000000000000000004cadb000000000000000000000000000000000000000000000000000000000000a604000000000000000000000000000000000000000000000000000000000001f4af00000000000000000000000000000000000000000000000000000000000053020000000000000000000000000000000000000000000000000000000000005302");
  }
}
```

Running `test_poc`:
```bash
  [128169] TitlesE2E::test_poc()
    ├─ [110871] 0x2ed6c4B5dA6378c7897AC67Ba9e43102Feb694EE::createSplit([0x077AAC78a05F5a116Fde61Cd4eF3081c175315c2, 0x133890338E33141727B2252Cc92F3081Ca212080, 0x1BCaff43A8607B0416d9e71586498A16C3486F03, 0x29d489e1c200740D91FCe85614a0a13376eda87A, 0x46761f1D443531afc94080E9F93d7aAd7C75aE87, 0x644E6abb3F9e8754bD9E07159F4Ee7d5E3c5c4Ac, 0x6d62D8e11b342D4D1f3054dD4C9D9540FEA76B3A, 0x87C926b07162BE81eD2Af05E3307a7D4A3F0AdE1, 0x8ed3e58Fcf1F2169F7B39Fc1AA8d753719EC53A8, 0x94d94eF8422418Fc5e8abA0DB5900a3454D8029E, 0xa05a94773154de8c0eeD5FDF7d1889Efea02Dc9A, 0xA7EcBb281fB3fE21943928282cb4452279b4A65c, 0xb5C1a33bBC95cE8541aF2e9C407F3989F9DABdd0, 0xb74f97dA97609ed9A592a12a4641Ebc5CD602406, 0xc01B3424b81da02A3aB626a44Cfe67c2eb15d9dA, 0xDb9ae004be5B96125685756c75B177189D5B47a6, 0xe3525546721893AA484dda2b7139Ba71D751be5e], [21250 [2.125e4], 106250 [1.062e5], 21250 [2.125e4], 42500 [4.25e4], 21250 [2.125e4], 110500 [1.105e5], 21250 [2.125e4], 21250 [2.125e4], 21250 [2.125e4], 42500 [4.25e4], 21250 [2.125e4], 22250 [2.225e4], 314075 [3.14e5], 42500 [4.25e4], 128175 [1.281e5], 21250 [2.125e4], 21250 [2.125e4]], 0, 0xc01B3424b81da02A3aB626a44Cfe67c2eb15d9dA)
    │   ├─ [18637] → new <unknown>@0xB49Cb08874f59eef91Fa323F9250a51817B4014D
    │   │   └─ ← [Return] 93 bytes of code
    │   ├─ emit CreateSplit(param0: 0xB49Cb08874f59eef91Fa323F9250a51817B4014D)
    │   └─ ← [Return] 0xB49Cb08874f59eef91Fa323F9250a51817B4014D
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← [Return]
    ├─ [5433] 0x2ed6c4B5dA6378c7897AC67Ba9e43102Feb694EE::createSplit([0x177aac78a05f5A116FDe61Cd4eF3081c175315C2, 0x133890338E33141727B2252Cc92F3081Ca212080, 0x1BCaff43A8607B0416d9e71586498A16C3486F03, 0x29d489e1c200740D91FCe85614a0a13376eda87A, 0x46761f1D443531afc94080E9F93d7aAd7C75aE87, 0x644E6abb3F9e8754bD9E07159F4Ee7d5E3c5c4Ac, 0x6d62D8e11b342D4D1f3054dD4C9D9540FEA76B3A, 0x87C926b07162BE81eD2Af05E3307a7D4A3F0AdE1, 0x8ed3e58Fcf1F2169F7B39Fc1AA8d753719EC53A8, 0x94d94eF8422418Fc5e8abA0DB5900a3454D8029E, 0xa05a94773154de8c0eeD5FDF7d1889Efea02Dc9A, 0xA7EcBb281fB3fE21943928282cb4452279b4A65c, 0xb5C1a33bBC95cE8541aF2e9C407F3989F9DABdd0, 0xb74f97dA97609ed9A592a12a4641Ebc5CD602406, 0xc01B3424b81da02A3aB626a44Cfe67c2eb15d9dA, 0xDb9ae004be5B96125685756c75B177189D5B47a6, 0xe3525546721893AA484dda2b7139Ba71D751be5e], [21250 [2.125e4], 106250 [1.062e5], 21250 [2.125e4], 42500 [4.25e4], 21250 [2.125e4], 110500 [1.105e5], 21250 [2.125e4], 21250 [2.125e4], 21250 [2.125e4], 42500 [4.25e4], 21250 [2.125e4], 22250 [2.225e4], 314075 [3.14e5], 42500 [4.25e4], 128175 [1.281e5], 21250 [2.125e4], 21250 [2.125e4]], 0, 0xc01B3424b81da02A3aB626a44Cfe67c2eb15d9dA)
    │   └─ ← [Revert]
    └─ ← [Stop]
```
First one succeeds - second one reverts because it's not ordered.
## Impact
Editions and Routes will not be able to be created, breaking the core functionality of this project.
## Code Snippet
[FeeManager.sol#L144](https://github.com/sherlock-audit/2024-04-titles/blob/c9d16782a7d3c15c7a759f22c9e0552d5e777ed7/wallflower-contract-v2/src/fees/FeeManager.sol#L144)
## Tool used
Manual Review
## Recommendation
Order the addresses before passing them to `createSplit()`.