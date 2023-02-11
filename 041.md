hyh

high

# UserManager's cancelVouchInternal breaks up voucher accounting on voucher removal

## Summary

_cancelVouchInternal() doesn't update `voucherIndices` array and incorrectly updates `vouchees` array (using `voucherIndexes` for `vouchees`, which are unrelated index-wise), messing up the vouchers accounting as a result.

## Vulnerability Detail

Currently _cancelVouchInternal() gets voucher array index from `voucherIndexes`, name it `voucheeIdx` and apply to `vouchees`, which isn't correct as those are different arrays, their indices are independent, so such addressing messes up the accounting data.

Also, proper update isn't carried out for voucher indices themselves, although it is required for the future referencing which is used in all voucher related activities of the protocol.

## Impact

Immediate impact is unavailability of vouch accounting logic for the borrower-staker combination that was this last element cancelVouch() moved.

Furthermore, if a new entry is placed, which is a high probability event being a part of normal activity, then old entry will point to incorrect staked/locked amount, and a various violations of the related constrains become possible.

Some examples are: trust can be updated via updateTrust() to be less then real locked amount; vouch cancellation can become blocked (say new voucher borrower gains a long-term lock and old lock with misplaced index cannot be cancelled until new lock be fully cleared, i.e. potentially for a long time).

The total impact is up to freezing the corresponding staker's funds as this opens up a way for various exploitations, for example a old entry's borrower can use the situation of unremovable trust and utilize this trust that staker would remove in a normal course of operations, locking extra funds of the staker this way.

The whole issue is a violation of the core UNION vouch accounting logic, with misplaced indices potentially piling up. Placing overall severity to be high.

## Code Snippet

_cancelVouchInternal() correctly treats `vouchee` entry removal update, but fails to do the same for `vouchers` case (first in the code):

https://github.com/sherlock-audit/2023-02-union/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L583-L620

```solidity
    function _cancelVouchInternal(address staker, address borrower) internal {
        Index memory removeVoucherIndex = voucherIndexes[borrower][staker];
        if (!removeVoucherIndex.isSet) revert VoucherNotFound();

        // Check that the locked amount for this vouch is 0
        Vouch memory vouch = vouchers[borrower][removeVoucherIndex.idx];
        if (vouch.locked > 0) revert LockedStakeNonZero();

        // Remove borrower from vouchers array by moving the last item into the position
        // of the index being removed and then poping the last item off the array
        {
            // Cache the last voucher
            Vouch memory lastVoucher = vouchers[borrower][vouchers[borrower].length - 1];
            // Move the lastVoucher to the index of the voucher we are removing
            vouchers[borrower][removeVoucherIndex.idx] = lastVoucher;
            // Pop the last vouch off the end of the vouchers array
            vouchers[borrower].pop();
            // Delete the voucher index for this borrower => staker pair
            delete voucherIndexes[borrower][staker];
            // Update the last vouchers coresponsing Vouchee item
            uint128 voucheeIdx = voucherIndexes[borrower][lastVoucher.staker].idx;
            vouchees[staker][voucheeIdx].voucherIndex = removeVoucherIndex.idx.toUint96();
        }

        // Update the vouchee entry for this borrower => staker pair
        {
            Index memory removeVoucheeIndex = voucheeIndexes[borrower][staker];
            // Cache the last vouchee
            Vouchee memory lastVouchee = vouchees[staker][vouchees[staker].length - 1];
            // Move the last vouchee to the index of the removed vouchee
            vouchees[staker][removeVoucheeIndex.idx] = lastVouchee;
            // Pop the last vouchee off the end of the vouchees array
            vouchees[staker].pop();
            // Delete the vouchee index for this borrower => staker pair
            delete voucheeIndexes[borrower][staker];
            // Update the vouchee indexes to the new vouchee index
            voucheeIndexes[lastVouchee.borrower][staker].idx = removeVoucheeIndex.idx;
        }
```

Namely, `voucherIndexes` to be updated as `lastVoucher` has been moved and entries for its old position to be corrected on the spot. `vouchees` to be updated as well, but using correct `voucheeIndexes` index reference.

## Tool used

Manual Review

## Recommendation

Consider setting both `voucherIndexes` and `vouchees` items for the last staker:

https://github.com/sherlock-audit/2023-02-union/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L591-L605

```solidity
        // Remove borrower from vouchers array by moving the last item into the position
        // of the index being removed and then poping the last item off the array
        {
            // Cache the last voucher
            Vouch memory lastVoucher = vouchers[borrower][vouchers[borrower].length - 1];
            // Move the lastVoucher to the index of the voucher we are removing
            vouchers[borrower][removeVoucherIndex.idx] = lastVoucher;
            // Pop the last vouch off the end of the vouchers array
            vouchers[borrower].pop();
            // Delete the voucher index for this borrower => staker pair
            delete voucherIndexes[borrower][staker];
-           // Update the last vouchers coresponsing Vouchee item
-           uint128 voucheeIdx = voucherIndexes[borrower][lastVoucher.staker].idx;
-           vouchees[staker][voucheeIdx].voucherIndex = removeVoucherIndex.idx.toUint96();
+           // Update the last vouchers coresponsing Vouchee item
+           vouchees[lastVoucher.staker][voucheeIndexes[borrower][lastVoucher.staker]].voucherIndex = removeVoucherIndex.idx.toUint96();
+           // Update the voucher indexes of the moved pair to the new voucher index
+           voucherIndexes[borrower][lastVoucher.staker].idx = removeVoucherIndex.idx;
        }
```