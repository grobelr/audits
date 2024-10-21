## M-01
Unspent allowance may break functionality in AMO

## Impact
An unspent allowance may cause a denial of service during the calls to safeApprove() in the AMO contract.

The AMO contract uses the safeApprove() function to grant the Uniswap pool permission to spend funds while adding liquidity, removing liquidity or swaping. When doing these functions into the Uniswap pool, the AMO contract needs to approve allowance so the AMM can pull tokens from the caller.

The safeApprove() function is a wrapper provided by the SafeERC20 library present in the OpenZeppelin contracts package, its implementation is the following:
```
function safeApprove(IERC20 token, address spender, uint256 value) internal {
    // safeApprove should only be called when setting an initial allowance,
    // or when resetting it to zero. To increase and decrease it, use
    // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
    require(
        (value == 0) || (token.allowance(address(this), spender) == 0),
        "SafeERC20: approve from non-zero to non-zero allowance"
    );
    _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
}
```
As the comment warns, this should only be used when setting an initial balance or resetting it to zero. In the AMO contract the use of safeApprove() is included in the functions that are in charge of adding liquidity/removing liquidity/swaping to the Uniswap pool, implying a repeatedly use whenever the allowance needs to be set so that the pool can pull the funds. As we can see in the implementation, if the current allowance is not zero the function will revert.

This means that any unspent allowance of tokens will cause a denial of service in the addLiquidity, removeLiquidity and swap functions, potentially bricking the contract.

## Proof of Concept
Suppose there is an unspent allowance of tokenA in the AMO contract, tokenA.allowance(UniV2LiquidityAMO, ammRounter) > 0.
DEFAULT_ADMIN_ROLE calls addLiquidity(tokenAAmount, tokenBAmount, tokenAAmountMin, tokenBAmountMin)
Transaction will be reverted in the call to safeApprove() as (value == 0) || (tokenA.allowance(address(this), spender) == 0) will be false.

## Tools Used
Manual Review, VS Code

## Recommended Mitigation Steps
Simply use approve(), or first reset the allowance to zero using safeApprove(spender, 0), or use safeIncreaseAllowance().
Also, use TransferHelper handler to do the approvals/transfers just like UniV3LiquidityAMO does.

## NOTE
Although this issue is highlighted in the automated findings under the identifier L-05, this report illustrates how this particular function could introduce a severe vulnerability that could potentially "brick" the contract. It's also worth noting that the L-05 issue report doesn't distinguish between libraries, leading to a misleading impression that the issue is of low severity.

The revised sentence aims to clarify the seriousness of the issue and correct any misunderstandings that might arise from the original automated findings.