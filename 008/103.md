Real Honeysuckle Bobcat

medium

# Token approval can be frontrun

## Summary

The GSP trading token approve() function can be frontrun such that a spender can double dip and spend an allowance more than once. 


## Vulnerability Detail

When a user sets a spender's allowance when their allowance is already non-zero, the spender can frontrun the user and double dip the allowance. This can occur through the following simple scenario:

1. Bob approves Alice 100 tokens as an allowance.
2. Bob decides to change Alice's allowance to 50 tokens.
3. Alice seeing Bob's tx decides to spend the entire allowance, receiving 100 tokens.
4. Bob's tx approval tx from Step 2 is executed and the allowance is now set to 50 tokens.
5. Alice spends this second allowance, receiving 50 tokens.

In this attack, Alice has received 150 tokens, 100 more than Bob desired.

## Impact

Third-party protocols and users who give approval to other users may experience front-running when approving tokens to other third-party entities and losing more than expected.

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol?plain=1#L265-L282

```solidity
/**
 * @dev Approve the passed address to spend the specified amount of tokens on behalf of msg.sender.
 * @param spender The address which will spend the funds.
 * @param amount The amount of tokens to be spent.
 */
function approve(address spender, uint256 amount) public returns (bool) {
    _approve(msg.sender, spender, amount);
    return true;
}

function _approve(
    address owner,
    address spender,
    uint256 amount
) private {
    _ALLOWED_[owner][spender] = amount;
}
```

## Tool used

Manual Review

## Recommendation

The protocol should implement a require check such that the approver can pass in a value to check what the current allowance value is:

```solidity
    function _approve(
        address owner,
        address spender,
        uint256 amount,
        uint256 expectedPreApproval
    ) private {
        require(_ALLOWED_[owner][spender] != expectedPreApproval, "Approval was frontrun");
        _ALLOWED_[owner][spender] = amount;
    }
```