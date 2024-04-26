Winning Scarlet Yeti

high

# Assets can be pulled and stolen from user's wallet

## Summary

Attackers can exploit the vulnerable `FeeManager` contract to steal assets from the user's wallet.

## Vulnerability Detail

The issue with the `FeeManager.collectCreationFee` and `FeeManager.collectMintFee` functions is that these functions are permissionless and can be executed by anyone. They also allow the caller to define an arbitrary `feePayer_`, which will lead to the issue described in the later section.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L166

```solidity
File: FeeManager.sol
166:     function collectCreationFee(IEdition edition_, uint256 tokenId_, address feePayer_)
167:         external
168:         payable
169:     {
170:         Fee memory fee = getCreationFee();
171:         if (fee.amount == 0) return;
172: 
173:         _route(fee, Target({target: protocolFeeReceiver, chainId: block.chainid}), feePayer_);
174:         emit FeeCollected(address(edition_), tokenId_, ETH_ADDRESS, fee.amount, 0);
175:     }
```

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L183

```solidity
File: FeeManager.sol
183:     function collectMintFee(
184:         IEdition edition_,
185:         uint256 tokenId_,
186:         uint256 amount_,
187:         address payer_,
188:         address referrer_
189:     ) external payable {
190:         _collectMintFee(
191:             edition_, tokenId_, amount_, payer_, referrer_, getMintFee(edition_, tokenId_, amount_)
192:         );
193:     }
```

Assume Bob grants the maximum allowance to the FeeManger contract or there is a residual allowance due to a previous transaction. 

Note that Bob has to grant the allowance to FeeManager because when he creates a new collection/work or mints a token, the `FeeManager.collectCreationFee` or `FeeManager.collectMintFee` function will attempt to pull the fee from the payer, which is pointed to Bob's wallet. If Bob does not grant the allowance to FeeManager, the pulling of tokens from Bob's wallet will fail and the transaction will revert.

Assume a malicious user called Alice is one of the fee recipients. She can call the `FeeManager.collectCreationFee` or `FeeManager.collectMintFee` function directly with the payer set to Bob's wallet address. The `FeeManager` will pull the assets from Bob's wallet and route the fee to Alice's wallet. She repeats this action multiple times until Bob's wallet is completely drained.

## Impact

Loss of assets for the victim as their wallets can be drained by attackers.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L166

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L183

## Tool used

Manual Review

## Recommendation

To remediate the issue, the following must be enforced:

1. The `FeeManager.collectCreationFee` and `FeeManager.collectMintFee` functions must not be permissionless. They should only be restricted to the `TitlesCore` and `Edition` contract, respectively.
2. The users/public should not be allowed to define the payer of the `FeeManager.collectCreationFee` and `FeeManager.collectMintFee` functions, and should not be given an option to choose the payer. The payer must be set to the caller (`msg.sender`) themselves within the creation and minting function.

Consider the following changes:

```diff
function collectCreationFee(IEdition edition_, uint256 tokenId_, address feePayer_)
    external
    payable
+		onlyTitlesCore    
{
    Fee memory fee = getCreationFee();
    if (fee.amount == 0) return;

    _route(fee, Target({target: protocolFeeReceiver, chainId: block.chainid}), feePayer_);
    emit FeeCollected(address(edition_), tokenId_, ETH_ADDRESS, fee.amount, 0);
}
```

```diff
function collectMintFee(
    IEdition edition_,
    uint256 tokenId_,
    uint256 amount_,
    address payer_,
    address referrer_
- ) external payable {
+ ) external payable onlyEditionContract {
    _collectMintFee(
        edition_, tokenId_, amount_, payer_, referrer_, getMintFee(edition_, tokenId_, amount_)
    );
}

function collectMintFee(
    IEdition edition,
    uint256 tokenId_,
    uint256 amount_,
    address payer_,
    address referrer_,
    Strategy calldata strategy_
- ) external payable {
+ ) external payable onlyEditionContract {
    _collectMintFee(
        edition, tokenId_, amount_, payer_, referrer_, getMintFee(strategy_, amount_)
    );
}
```

In addition, ensure that all contracts that execute the `FeeManager.collectCreationFee` and `FeeManager.collectMintFee` functions set the payer to the caller (`msg.sender`) under all scenarios.