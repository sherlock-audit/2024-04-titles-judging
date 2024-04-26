Winning Scarlet Yeti

medium

# Malicious users can block creators from acknowledging or deacknowledging an edge

## Summary

Malicious users can block someone from acknowledging or deacknowledging an edge, affecting the sanctity of the data in the Graph.

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

Both `acknowledgeEdge` and `unacknowledgeEdge` functions rely on the same modifier (`checkSignature`) to verify the signature validity. Thus, the signature used for acknowledgment and deacknowledgment of an edge follows the same format and can be used interchangeably. However, this design creates an issue, as described next.

1. Assume that Bob wants to acknowledge an edge. Thus, Bob calls the `acknowledgeEdge` function with his signature $Sig_1$.

2. A malicious user can always front-run Bob, take his signature ($Sig_1$) and sent to the `unacknowledgeEdge` function instead. When the `unacknowledgeEdge` function is executed with $Sig_1$, the signature will be marked as used by the code `_isUsed[keccak256(signature)] = true;`.

3. When Bob's `acknowledgeEdge` transaction gets executed, it will revert because his signature ($Sig_1$) has already been used.

The malicious users can keep repeating as the chain's gas fees on L2 are cheap.

The same trick can also be used to block someone from deacknowledge an edge.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L118

```solidity
File: TitlesGraph.sol
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

Malicious users can block someone from acknowledging or deacknowledging an edge, affecting the sanctity of the data in the Graph.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L118

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L146

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L40

## Tool used

Manual Review

## Recommendation

```solidity
bytes32 public constant ACK_TYPEHASH = keccak256("Ack(bytes32 edgeId,bytes data)");
bytes32 public constant DEACK_TYPEHASH = keccak256("Deack(bytes32 edgeId,bytes data)");

bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
bytes32 digest = _hashTypedData(keccak256(abi.encode(DEACK_TYPEHASH, edgeId, data)));
```

Consider using two different hash types for acknowledging or deacknowledging within the signature and use a different modifier for checking the signature. This will prevent malicious users from taking the signature intended for `acknowledgeEdge` and submitting it to `unacknowledgeEdge`, and vice versa.

```solidity
modifier checkSignatureForAck(bytes32 edgeId, bytes calldata data, bytes calldata signature) {
    bytes32 digest = _hashTypedData(keccak256(abi.encode(ACK_TYPEHASH, edgeId, data)));
    ..SNIP..
}

modifier checkSignatureForDeack(bytes32 edgeId, bytes calldata data, bytes calldata signature) {
    bytes32 digest = _hashTypedData(keccak256(abi.encode(DEACK_TYPEHASH, edgeId, data)));
    ..SNIP..
}
```

```diff
function acknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
    external
-   checkSignature(edgeId_, data_, signature_)
+   checkSignatureForAck(edgeId_, data_, signature_)
    returns (Edge memory edge)
{

function unacknowledgeEdge(bytes32 edgeId_, bytes calldata data_, bytes calldata signature_)
    external
-   checkSignature(edgeId_, data_, signature_)
+		checkSignatureForDeack(edgeId_, data_, signature_)
    returns (Edge memory edge)
{
```

With this design, if a creator creates a signature intended for `acknowledgeEdge` function, and a malicious user front-runs and copies the signature and submits it to `acknowledgeEdge` function, no harm is done as the malicious user is simply executing the task on behalf of the creator. The edge will still be acknowledged at the end.