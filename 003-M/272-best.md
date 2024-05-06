Winning Scarlet Yeti

medium

# Uninitialized `TitlesCore` implementation contract can be taken over by an attacker

## Summary

An attacker could take over the implementation/logic contract of  `TitlesCore`, which might impact the proxy.

## Vulnerability Detail

Per Line 21, the `TitlesCore` contract inherits from the `UUPSUpgradeable` contract. The `TitlesCore` contract will contain the logic/implementation, and the UUPS proxy will point its implementation to the `TitlesCore` contract address.

It was observed that the `TitlesCore` implementation/logic contract is left uninitialized. As a result, an attacker could take over the implementation/logic contract of  `TitlesCore` by calling the `TitlesCore.initialize` function directly on the `TitlesCore` implementation/logic contract, which might impact the proxy.

When that happens, the attackers will become the owner of the implementation/logic contract per Line 45 below.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L45

```solidity
File: TitlesCore.sol
32: contract TitlesCore is OwnableRoles, Initializable, UUPSUpgradeable, Receiver {
..SNIP..
44:     function initialize(address feeReceiver_, address splitFactory_) external initializer {
45:         _initializeOwner(msg.sender);
46: 
47:         feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
48:         graph = new TitlesGraph(address(this), msg.sender);
49:     }
```

## Impact

An attacker could take over the implementation/logic contract of  `TitlesCore`, which might impact the proxy.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L45

## Tool used

Manual Review

## Recommendation

To prevent the implementation contract from being used or taken over, invoke the `initializer` in the constructor to automatically lock the `initializer` on the implementation contract when it is deployed. This is also the recommendation from OpenZeppelin when handling upgradable contracts (https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#initializing_the_implementation_contract).

```diff
contract TitlesCore is OwnableRoles, Initializable, UUPSUpgradeable, Receiver {
    using LibClone for address;
    using LibZip for bytes;
    using SafeTransferLib for address;

    address public editionImplementation = address(new Edition());
    FeeManager public feeManager;
    TitlesGraph public graph;

+		constructor() initializer {}

    /// @notice Initializes the protocol.
    /// @param feeReceiver_ The address to receive fees.
    /// @param splitFactory_ The address of the split factory.
    function initialize(address feeReceiver_, address splitFactory_) external initializer {
        _initializeOwner(msg.sender);

        feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
        graph = new TitlesGraph(address(this), msg.sender);
    }
```

Ensure that this change is also applied to the `TitlesGraph` contract, as it is also an upgradable contract.