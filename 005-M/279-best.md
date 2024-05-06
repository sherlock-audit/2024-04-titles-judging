Winning Scarlet Yeti

medium

# Signature is malleable

## Summary

The signature is malleable. When a signature is malleable, it means that it is possible to produce another valid signature for the same message (which also means the same digest).

If the intention is only to allow the creator to acknowledge an edge once, the creator can bypass this restriction because the signature is malleable, and the creator can create another signature to acknowledge an edge again.

## Vulnerability Detail

The protocol relies on Solady's [`SignatureCheckerLib`](https://github.com/vectorized/solady/blob/main/src/utils/SignatureCheckerLib.sol) to verify that the signature provided is valid. Once a signature has been successfully verified, it will be marked as used in Line 49 below to prevent users from re-using or replaying the same signature. 

The Solady's SignatureCheckerLib warns that the library does not check if a signature is non-malleable.

https://github.com/Vectorized/solady/blob/a34977e56cc1437b7ac07e6356261d2b303da686/src/utils/SignatureCheckerLib.sol#L23

```solidity
/// WARNING! Do NOT use signatures as unique identifiers:
/// - Use a nonce in the digest to prevent replay attacks on the same contract.
/// - Use EIP-712 for the digest to prevent replay attacks across different chains and contracts.
///   EIP-712 also enables readable signing of typed data for better user safety.
/// This implementation does NOT check if a signature is non-malleable.
library SignatureCheckerLib {
```

Based on the following code, a creator can only acknowledge an edge once because the digest of the signature to acknowledge an edge will be the same (assuming that `data` is not in use) every time. After a creator acknowledges an edge, the signature (which also means its digest) will be marked as used in Line 49 below, thus preventing the creator from acknowledging the edge again later using the same signature.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L49

```solidity
File: TitlesGraph.sol
39:     /// @notice Modified to check the signature for a proxied acknowledgment.
40:     modifier checkSignature(bytes32 edgeId, bytes calldata data, bytes calldata signature) {
41:         bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
42:         if (
43:             !edges[edgeId].to.creator.target.isValidSignatureNowCalldata(digest, signature)
44:                 || _isUsed[keccak256(signature)]
45:         ) {
46:             revert Unauthorized();
47:         }
48:         _;
49:         _isUsed[keccak256(signature)] = true;
50:     }
```

## Impact

When a signature is malleable, it means that it is possible to produce another valid signature for the same message (which also means the same digest).

If the intention is only to allow the creator to acknowledge an edge once, the creator can bypass this restriction because the signature is malleable, and the creator can create another signature to acknowledge an edge again.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L49

## Tool used

Manual Review

## Recommendation

Consider verifying the `s` of the signature is within valid bounds to avoid signature malleability.

```solidity
if (uint256(s) > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0) {
	 revert("ECDSA: invalid signature 's' value");
}
```