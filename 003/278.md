Winning Scarlet Yeti

medium

# Edition implementation not initialized on proxy

## Summary

Edition implementation is not initialized on the proxy. As a result, the ability to create an edition, which is a core functionality of the protocol, is broken.

## Vulnerability Detail

The `TitlesCore` is the logic/implementation contract behind the ERC1967 proxy. The protocol adopts the UUPS proxy design.

It was observed that there is an issue at Line 37 below. When using a proxy contract, any direct initialization of state variables in the implementation contract (like `address public editionImplementation = address(new Edition());` directly in the field declaration) only affects the storage layout at the implementation level during deployment and does not affect the proxy’s state.

As a result, on the proxy itself, the `editionImplementation` will remain uninitialized, which is default to the `0x0000000000000000000000000000000000000000` (the zero address).

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L37

```solidity
File: TitlesCore.sol
32: contract TitlesCore is OwnableRoles, Initializable, UUPSUpgradeable, Receiver {
33:     using LibClone for address;
34:     using LibZip for bytes;
35:     using SafeTransferLib for address;
36: 
37:     address public editionImplementation = address(new Edition());
38:     FeeManager public feeManager;
39:     TitlesGraph public graph;
40: 
41:     /// @notice Initializes the protocol.
42:     /// @param feeReceiver_ The address to receive fees.
43:     /// @param splitFactory_ The address of the split factory.
44:     function initialize(address feeReceiver_, address splitFactory_) external initializer {
45:         _initializeOwner(msg.sender);
46: 
47:         feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
48:         graph = new TitlesGraph(address(this), msg.sender);
49:     }
```

When someone creates a new Edition, the Edition contract will be cloned from the `editionImplementation` as shown at Line 79 below. Since the `editionImplementation` is not initialized, the clone will fail and revert. As a result, users will not be able to create an edition.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L79

```solidity
File: TitlesCore.sol
68:     /// @notice Creates an {Edition} with the given payload.
69:     /// @param payload_ The compressed payload for creating the {Edition}. See {EditionPayload}.
70:     /// @param referrer_ The address of the referrer.
71:     /// @return edition The new {Edition}.
72:     function createEdition(bytes calldata payload_, address referrer_)
73:         external
74:         payable
75:         returns (Edition edition)
76:     {
77:         EditionPayload memory payload = abi.decode(payload_.cdDecompress(), (EditionPayload));
78: 
79:         edition = Edition(editionImplementation.clone());
```

## Impact

Creating an edition is a core functionality of the protocol and this function is broken. 

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L37

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L79

## Tool used

Manual Review

## Recommendation

Consider the following changes to ensure that the `editionImplementation` is initialized properly on the proxy.

```diff
- address public editionImplementation = address(new Edition());
+ address public editionImplementation;

function initialize(address feeReceiver_, address splitFactory_) external initializer {
    _initializeOwner(msg.sender);

    feeManager = new FeeManager(msg.sender, feeReceiver_, splitFactory_);
    graph = new TitlesGraph(address(this), msg.sender);
+   editionImplementation = address(new Edition());
}
```