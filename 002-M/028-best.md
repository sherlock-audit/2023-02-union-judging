mrpathfindr

medium

# Possible Reentrancy: State Variable should be updated before token transfer.

## Summary

Possible Reentrancy /Unnecessary double execution of in Comptroller.sol.withdrawRewards()

The state variable ```accrued``` used to calculate rewards in  ```Comptroller_calculateRewardsByBlocks()``` is updated after ```safeTransfer()```  in  ```Comptroller_withdrawRewards()```. 


Link to code: https://github.com/sherlock-audit/2023-02-union/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L247

## Vulnerability Detail

An attacker could essentially call  ```_withdrawRewards()``` multiple times since the  ```_calculateRewardsByBlocks``` return value will not  be  updated here ```users[account][token].accrued = 0;``` until after  ```unionToken.safeTransfer(account, amount);``` is executed. 

If the address of the attacker specified in  ```users[account]``` is a malicious contract, a callback function can be triggered to reenter the  ```_withdrawRewards()``` function.

## Impact

Possible miscalculation of  ```accrued``` value could cause the if condition  ```if (unionToken.balanceOf(address(this)) >= amount && amount > 0) ``` to be true since  ```amount = _calculateRewardsByBlocks()``` where  ```_calculateRewardsByBlocks()``` returns 
            ```userInfo.accrued +
            (curInflationIndex - startInflationIndex).wadMul(user.effectiveStaked).wadMul(rewardMultiplier);```

Causing the code block within the if condition to be executed again. 
            
            
This could cause an unintended transfer of tokens or an unnecessary execution of the if condition previously mentioned. 

## Code Snippet

```solidity
function withdrawRewards(address account, address token) external override whenNotPaused returns (uint256) {
        IUserManager userManager = _getUserManager(token);

        // Lookup account state from UserManager
        (UserManagerAccountState memory user, Info memory userInfo, uint256 pastBlocks) = _getUserInfo(
            userManager,
            account,
            token,
            0
        );

        // Lookup global state from UserManager
        uint256 globalTotalStaked = userManager.globalTotalStaked();

        uint256 amount = _calculateRewardsByBlocks(account, token, pastBlocks, userInfo, globalTotalStaked, user);

        // update the global states
        gInflationIndex = _getInflationIndexNew(globalTotalStaked, block.number - gLastUpdatedBlock);
        gLastUpdatedBlock = block.number;
        users[account][token].updatedBlock = block.number;
        users[account][token].inflationIndex = gInflationIndex;
        if (unionToken.balanceOf(address(this)) >= amount && amount > 0) {
            unionToken.safeTransfer(account, amount);
            users[account][token].accrued = 0;
            emit LogWithdrawRewards(account, amount);

            return amount;
        } else {
            users[account][token].accrued = amount;
            emit LogWithdrawRewards(account, 0);

            return 0;
        }
    } 
```

## Tool used

Manual Review

## Recommendation

Always update state variables before token transfers.


```solidity
if (unionToken.balanceOf(address(this)) >= amount && amount > 0) {
        users[account][token].accrued = 0;
        unionToken.safeTransfer(account, amount);
        emit LogWithdrawRewards(account, amount);

        return amount;
    } else {
        users[account][token].accrued = amount;
        emit LogWithdrawRewards(account, 0);

        return 0;
    } 
```
