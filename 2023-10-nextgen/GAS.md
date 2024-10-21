## 1. Contrato XRandoms.sol
 In the function randomWord() there is the same random number implementation used in the function randomNumber(), instead of making 2 implementations, call the getRandomNumber function to save on gas cost. 
    - original gas cost estimate function = 400 a 650.
    - function optimization = 200 a 400.
```
        function randomNumber() public view returns (uint256){
            uint256 randomNum = uint(keccak256(abi.encodePacked(block.prevrandao, blockhash(block.number - 1), block.timestamp))) % 1000;
            return randomNum;
        }

    -    function randomWord() public view returns (string memory) {
    -        uint256 randomNum = uint(keccak256(abi.encodePacked(block.prevrandao, blockhash(block.number - 1), block.timestamp))) % 100;
    -        return getWord(randomNum);
    +        function randomWord() public view returns (string memory) {
    +        return getWord(randomNumber());
        }
``` 

In the getWord function, there is 1 conditional if it is equal to zero it returns the index sent, if it does not return the index - 1, running the risk of bringing the index to zero more than once.

        function getWord(uint256 id) private pure returns (string memory) {
    -
            // array storing the words list
            string[100] memory wordsList = ["Acai", "Ackee", "Apple", "Apricot", "Avocado", "Babaco", "Banana", "Bilberry", "Blackberry", "Blackcurrant", "Blood Orange",
            "Blueberry", "Boysenberry", "Breadfruit", "Brush Cherry", "Canary Melon", "Cantaloupe", "Carambola", "Casaba Melon", "Cherimoya", "Cherry", "Clementine",
    @@ -25,25 +24,17 @@ contract randomPool {
            "Soursop", "Star Apple", "Star Fruit", "Strawberry", "Sugar Apple", "Tamarillo", "Tamarind", "Tangelo", "Tangerine", "Ugli", "Velvet Apple", "Watermelon"];

            // returns a word based on index
    -        if (id==0) {
    +        
                return wordsList[id];
    -        } else {
    -            return wordsList[id - 1];
    -        }
    -        }
    +      
    +    }

The returnIndex function is not necessary, just call the getword() function directly.
    -
    -    function returnIndex(uint256 id) public view returns (string memory) {
    -        return getWord(id);
    -    }
    -

## 2. AuctionDemo.sol
In the returnHighestBid() function, if we declared the variable outside the scope of the first conditional and just returned the state of the highBid variable, we have an optimization in the gas cost of approximately 1200.
        function returnHighestBid(uint256 _tokenid) public view returns (uint256) {
            uint256 index;
    +        uint256 highBid = 0;
            if (auctionInfoData[_tokenid].length > 0) {
    -            uint256 highBid = 0;
    -            for (uint256 i=0; i< auctionInfoData[_tokenid].length; i++) {
    -                if (auctionInfoData[_tokenid][i].bid > highBid && auctionInfoData[_tokenid][i].status == true) {
    +            for (uint256 i = 0; i < auctionInfoData[_tokenid].length; i++) {
    +                if (
    +                    auctionInfoData[_tokenid][i].bid > highBid &&
    +                    auctionInfoData[_tokenid][i].status == true
    +                ) {
                        highBid = auctionInfoData[_tokenid][i].bid;
                        index = i;
                    }
                }
    -            if (auctionInfoData[_tokenid][index].status == true) {
    -                return highBid;
    -            } else {
    -                return 0;
    -            }
    -        } else {
    -            return 0;
            }
    +        return highBid;
        }

    base 20 array lenght
    optimization = 300;  
    original = 1500; 

Removing the highbid variable and replacing the if with a require in the second implementation of the returnHighestBidder function can save up to 50% of gas compared to the first implementation.
        function returnHighestBidder(uint256 _tokenid) public view returns (address) {
    -        uint256 highBid = 0;
            uint256 index;
    -        for (uint256 i=0; i< auctionInfoData[_tokenid].length; i++) {
    -            if (auctionInfoData[_tokenid][i].bid > highBid && auctionInfoData[_tokenid][i].status == true) {
    +        for (uint256 i = 0; i < auctionInfoData[_tokenid].length; i++) {
    +            if (
    +                auctionInfoData[_tokenid][i].bid > 0 &&
    +                auctionInfoData[_tokenid][i].status == true
    +            ) {
                    index = i;
                }
            }
    -        if (auctionInfoData[_tokenid][index].status == true) {
    -                return auctionInfoData[_tokenid][index].bidder;
    -            } else {
    -                revert("No Active Bidder");
    -        }
    +        require(!auctionInfoData[_tokenid][index].status == true,"No Active Bidder");
    +        return auctionInfoData[_tokenid][index].bidder;
        }
        
    optimization = 640
    original = 1400

Remove an unused else block
    @@ -115,7 +111,7 @@ contract auctionDemo is Ownable {
                } else if (auctionInfoData[_tokenid][i].status == true) {
                    (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}("");
                    emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
    -            } else {}
    +            }
            }
        } 

 ## 3. MinterContract.sol 
 Declaring the Tdiff variable in the scope of function execution can save gas, as it avoids memory allocation if the price calculation flow is not the same as when using Tdiff in the function.

      function getPrice(uint256 _collectionId) public view returns (uint256) {
-        uint tDiff;
+       
         if (collectionPhases[_collectionId].salesOption == 3) {
             // increase minting price by mintcost / collectionPhases[_collectionId].rate every mint (1mint/period)
             // to get the price rate needs to be set
@@ -543,7 +543,7 @@ contract NextGenMinterContract is Ownable {
             // if just public mint set the publicStartTime = allowlistStartTime
             // if rate = 0 exponetialy decrease
             // if rate is set the linear decrase each period per rate
-            tDiff = (block.timestamp - collectionPhases[_collectionId].allowlistStartTime) / collectionPhases[_collectionId].timePeriod;
+            uint tDiff = (block.timestamp - collectionPhases[_collectionId].allowlistStartTime) / collectionPhases[_collectionId].timePeriod;
             uint256 price;
             uint256 decreaserate;
             if (collectionPhases[_collectionId].rate == 0) {

## 4. RandomizerNXT.sol
Remove unused parameter calculateTokenHash() _saltfun_o
index 0c7818c..b6d310d 100644
    --- a/smart-contracts/RandomizerNXT.sol
    +++ b/smart-contracts/RandomizerNXT.sol
    @@ -52,7 +52,9 @@ contract NextGenRandomizerNXT {
        }

        // function that calculates the random hash and returns it to the gencore contract
    -    function calculateTokenHash(uint256 _collectionID, uint256 _mintIndex, uint256 _saltfun_o) public {
    +    function calculateTokenHash(uint256 _collectionID, uint256 _mintIndex) public {  
    +
    +        
            require(msg.sender == gencore);
            bytes32 hash = keccak256(abi.encodePacked(_mintIndex, blockhash(block.number - 1), randoms.randomNumber(), randoms.randomWord()));
            gencoreContract.setTokenHash(_collectionID, _mintIndex, hash);
    diff --git a/smart-contracts/XRandoms.sol b/smart-contracts/XRandoms.sol