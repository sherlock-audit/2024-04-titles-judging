Winning Scarlet Yeti

medium

# Creator cannot acknowledge or deacknowledge an edge twice

## Summary

The creator cannot acknowledge or deacknowledge an edge twice, affecting the sanctity of the data in the Graph.

## Vulnerability Detail

> [!IMPORTANT]
>
> The following is an extract from the [contest's README](https://audits.sherlock.xyz/contests/326):
>
> > Additional audit information.
> >
> > In addition to the security of funds, we would also like there to be focus on the sanctity of the data in the TitlesGraph and the permissioning around it (only the appropriate people/contracts can signal reference and acknowledgement of reference).
>
> The contest's README stated that apart from the loss of assets, the protocol team would like there to be a focus on the sanctity of the data. Thus, any issues related to the sanctity of the data in the Graph would be considered a valid Medium issue in the context of this audit contest, as the Contest's README supersede Sherlock's judging rules per Sherlock's Hierarchy of truth.

Both `acknowledgeEdge` and `unacknowledgeEdge` functions rely on the same modifier (`checkSignature`) to verify the signature validity. Thus, the signature used for acknowledgment and deacknowledgment of an edge follows the same format and can be used interchangeably. 

Assume that the `data_` is not currently in use and set to empty/null. Thus, the signature's digest to acknowledge and deacknowledge an Edge (edgeId=800) is as follows, which is exactly the same.

```solidity
bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
```

Assume that Bob wants to acknowledge an EdgeID of 800. His signature digest will be as follows:

```solidity
digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, 800, 0)));
```

After Bob has executed the `acknowledgeEdge` function with the signature, the above digest will be marked as used, and cannot be used for a second time.

Let's assume some time has passed, and Bob decided to deacknowledge EdgeID of 800. In this case, his signature digest will be as follows:

```solidity
digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, 800, 0)));
```

The above digest is exactly the same digest that had been marked as used earlier. Thus, Bob's attempts to deacknowledge EdgeID of 800 will fail. 

The same issue will also occur in many scenarios, such as (ack > deack > ack). In short, this issue will occur when acknowledgment and deacknowledgment are called for a second time with a signature.

The root cause of this issue is that the data hashed within the digest lacks the nonce or certain data that is unique across different signatures.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L118

```solidity
File: TitlesGraph.sol
112:     /// @notice Acknowledge an edge using an ECDSA signature.
113:     /// @param edgeId_ The ID of the edge to acknowledge.
114:     /// @param data_ Additional data to include with the acknowledgment.
115:     /// @param signature_ The ECDSA signature to verify.
116:     /// @dev The request is valid if the given signature was produced using the edge ID as the message and the creator of the `to` node as the signer.
117:     /// @dev This function is used to acknowledge an edge that was previously created and will revert if the edge does not exist or if the signature is invalid. It emits an {EdgeAcknowledged} event for the edge.
118:     function acknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
119:         external
120:         checkSignature(edgeId_, data_, signature_)
121:         returns (Edge memory edge)
122:     {
123:         return _setAcknowledged(edgeId_, data_, true);
124:     }
```
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L146
```solidity
File: TitlesGraph.sol
140:     /// @notice Unacknowledge an edge using an ECDSA signature.
141:     /// @param edgeId_ The ID of the edge to unacknowledge.
142:     /// @param data_ Additional data to include with the unacknowledgment.
143:     /// @param signature_ The ECDSA signature to verify.
144:     /// @dev The request is valid if the given signature was produced using the edge ID as the message and the creator of the `to` node as the signer.
145:     /// @dev This function is used to unacknowledge an edge that was previously acknowledged and will revert if the edge does not exist or if the signature is invalid. It emits an {EdgeUnacknowledged} event for the edge.
146:     function unacknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
147:         external
148:         checkSignature(edgeId_, data_, signature_)
149:         returns (Edge memory edge)
150:     {
151:         return _setAcknowledged(edgeId_, data_, false);
152:     }
```
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L40
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

The creator cannot acknowledge or deacknowledge an edge twice, affecting the sanctity of the data in the Graph.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L118

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L146

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L40

## Tool used

Manual Review

## Recommendation

The standard security practice is to include a nonce into the signature digest to ensure its uniqueness and prevent potential replay attacks or collisions.

Consider adding a nonce to the digest and incrementing the creator's nonce after each successful execution of the acknowledgment or deacknowledgment.

```diff
- bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
+ bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data, nonce[creator])));
```