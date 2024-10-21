## M-01 Lack of Functionality to Transfer `delegatedAddress` Role

The contract [FigtherFarmer](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L54) doesn't have a function that allows for the transfer or update of the `delegatedAddress` to another address after the contract has been deployed.

### Impact

The inability to update the `delegatedAddress` post-deployment could lead to operational and security issues:

1. **Operational Inflexibility**: If the `delegatedAddress`'s private key is compromised or there's a need to change the entity responsible for message signing, the contract lacks the necessary functionality to update the address, potentially rendering any dependent functionality inoperative or insecure.

2. **Security Risk**: In the event the `delegatedAddress` is compromised, the contract administrator cannot revoke the compromised address's access, potentially allowing unauthorized actions.

### Recommendation

Implement a function that enables the contract owner or a designated administrator to update the `delegatedAddress`. This function should include appropriate security measures such as access controls and, ideally, emit an event for transparency. Ensuring that only authorized individuals can execute this function is crucial to prevent unauthorized changes.

#### Example Implementation

```solidity
/// @notice Allows the owner to set a new delegated address.
/// @param newDelegatedAddress The new address to be delegated.
function setDelegatedAddress(address newDelegatedAddress) public {
    require(msg.sender == _ownerAddress);
    require(newDelegatedAddress != address(0), "Invalid address");   
    _delegatedAddress = newDelegatedAddress;
    emit DelegatedAddressChanged(newDelegatedAddress);
}
```

## M-02 Absence of Role Revocation Mechanism

The contract Neuron.sol the functions to add addresses to specific roles [minter](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L93), [staker](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L101) and [spender](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L109), there is no corresponding functionality provided to revoke these roles once assigned. 

In the Neuron.sol contract, while there are functions provided for assigning specific roles [minters](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L93), [stakers](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L101), and [spenders](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L109) addresses, the contract lacks the essential mechanisms for revoking these roles once they have been granted. This omission limits the contract's ability to adapt to changes or respond to security concerns by removing roles from addresses as necessary.

### Impact

The inability to revoke roles from addresses can lead to several issues:

1. **Operational Rigidity**: Once an address is assigned a role, the contract lacks the flexibility to adapt to changes such as role reallocation, addressing compromised security, or merely updating the set of addresses holding a specific role.

3. **Contract Management Limitations**: Without the ability to revoke roles, the contract's management becomes cumbersome, particularly in dynamic environments where changes to roles are expected. This limitation may necessitate deploying a new contract or implementing additional layers of management logic outside the smart contract to handle role assignments dynamically.

### Recommendation

It's recommended to implement corresponding functions to revoke each role (`minter`, `staker`, and `spender`) that can be called by the contract owner or another designated authority within the contract. These functions should ensure that only authorized parties can remove roles, with adequate checks.

#### Example Implementation

```solidity
/// @notice Revokes a minter role from an address.
/// @param minterAddress The address to be removed from the minter role.
function revokeMinter(address minterAddress) external {
    require(msg.sender == _ownerAddress);
    revokeRole(MINTER_ROLE, minterAddress);
    emit RoleRevoked(MINTER_ROLE, minterAddress, msg.sender);
}

/// @notice Revokes a staker role from an address.
/// @param stakerAddress The address to be removed from the staker role.
function revokeStaker(address stakerAddress) external {
    require(msg.sender == _ownerAddress);
    revokeRole(STAKER_ROLE, stakerAddress);
    emit RoleRevoked(STAKER_ROLE, stakerAddress, msg.sender);
}

/// @notice Revokes a spender role from an address.
/// @param spenderAddress The address to be removed from the spender role.
function revokeSpender(address spenderAddress) external {
    require(msg.sender == _ownerAddress);
    revokeRole(SPENDER_ROLE, spenderAddress);
    emit RoleRevoked(SPENDER_ROLE, spenderAddress, msg.sender);
}
```

## H-01 Potential Reentrancy in `_createNewFighter` Function

The [_createNewFighter](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L212) function may be susceptible to forcible revert attacks due to the call to `_safeMint` within its body. Since `_safeMint` triggers the ERC721 `Transfer` event, if the recipient address is a contract, it will invoke the `onERC721Received` function on the recipient contract.

### Impact

The design of this function introduces a potential exploit vector related to the [`reRoll`](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L370) function, which allows users to spend $NRN tokens to reroll the attributes of their fighters. Malicious actors or contracts could exploit the ability to forcibly revert transactions to avoid the cost associated with rerolling attributes until they obtain a fighter with desired attributes. 

### POC 
Modify the onERC721Received function in the Foundry test file [FighterFarm.t.sol](https://github.com/code-423n4/2024-02-ai-arena/blob/main/test/FighterFarm.t.sol#L404)  as follows:
```
function onERC721Received(address, address, uint256, bytes memory) public returns (bytes4) {
        // Handle the token transfer here
        (
            address owner,
            uint256[6] memory attr,
            uint256 weight,
            uint256 element,
            ,
            ,
            uint16 generation
        ) = _fighterFarmContract.getAllFighterInfo(0);

        // Revert the transaction forcibly if the attributes do not meet specific 
        // criteria to prevent spending $NRN tokens on an unwanted random reroll outcome.
        if (attr[1] != 1) {
            return bytes4(0x0);
        } else {
            return this.onERC721Received.selector;
        }
    }
```
And now run forge tests: 
```
forge test --mt testReroll -vv

Failing tests:
Encountered 2 failing tests in test/FighterFarm.t.sol:FighterFarmTest
[FAIL. Reason: ERC721: transfer to non ERC721Receiver implementer] testReroll() (gas: 444802)
[FAIL. Reason: ERC721: transfer to non ERC721Receiver implementer] testRerollRevertWhenLimitExceeded() (gas: 444822)
```

### Recommendation

Require users to commit to the attributes of the fighter or the outcome of the `reRoll` before the minting transaction, reducing the incentive to revert the transaction after seeing the results.
