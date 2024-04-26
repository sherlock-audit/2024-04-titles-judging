Big Stone Wren

medium

# Deployment to several chains would fail due to incorrect Solidity version

## Summary
The TITLES protocol uses Solidity version `0.8.24` which would lead to deployment failure on Base, Zora, Blast, zkSync and Degen

## Vulnerability Detail
Solidity `v0.8.20` introduced a new opcode `PUSH0`. Meanwhile this does not cause an issue for deployment to the Ethereum network, `PUSH0` opcode is not yet supported by many chains. The contract bytecode would have a `PUSH0` opcode which would revert on deploymeny to all chains except Ethereum, Optimism and Arbitrum.

```diff
+ Ethereum - Supported
- Base - Not Supported (Fork of OP Stack Bedrock version; Canyon version and further support `PUSH0` opcode)
+ Optimism - Supported
- Zora - Not Supported (Fork of OP Stack Bedrock version; Canyon version and further support `PUSH0` opcode)
- Blast - Not Supported (Fork of OP Stack Bedrock version; Canyon version and further support `PUSH0` opcode)
+ Arbitrum - Supported (Make sure to use a node provider like Chainstack who have upgraded nodes to ArbOS11 or later)
- zkSync - Not Supported
- Degen - Not Supported (L3 built using Arbitrum Orbit, which utilizes Arbitrum Nitro which is a fork for Geth)

```

## Impact
Restricts the ability of the protocol to deploy on multiple chains.

## Code Snippet
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L2
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L2
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L2
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L2
https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/shared/Common.sol#L2

## Tool used
Manual Review
Documentation

## Recommendation
Downgrade to Solidity `v0.8.19`