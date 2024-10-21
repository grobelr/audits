## M-01 StakedUSDe::redistributeLockedAmount is not checking _checkMinShares in burn
The function [redistributeLockedAmount](https://github.com/code-423n4/2023-10-ethena/blob/main/contracts/StakedUSDe.sol#L148) is not checking if after the burn small non-zero amount of shares does not remain, exposing to donation attack.

## Impact
The absence of a _checkMinShares check in the redistributeLockedAmount function, when the to address is set to 0 (indicating a burn), could lead to a situation where a small, non-zero amount of shares remains post-burn. This scenario is problematic as it contradicts the protocol’s preventative measures against donation attacks, implemented in deposit and withdraw functions.

## Proof of Concept
There's already one test here:
[StakedUSDe.balcklist.t.sol](https://github.com/code-423n4/2023-10-ethena/blob/main/test/foundry/staking/StakedUSDe.blacklist.t.sol#L335) 
The suggested addition of `assertGt(stakedUSDe.totalSupply(), 1 ether);` at the end of the function would serve to validate the presence of the issue. (simple and clear, no need to rewrite POCs here)

## Tools Used
Foundry, VS Code

## Recommended Mitigation Steps:
Add _checkMinShares() when the function redistributeLockedAmoun is called with to = 0.