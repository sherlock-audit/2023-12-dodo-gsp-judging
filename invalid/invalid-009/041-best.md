Smooth Wool Buffalo

medium

# Shares can be minted to 0 address

## Summary
No zero address checks in the `_mint` and `buyShares` function causes that shares can be minted to anon existent address. 
## Vulnerability Detail

The `buyShares` function doesn't check for `to` address existence, causing that shares can be minted to a zero address.

```solidity
    function buyShares(address to)
        external
        nonReentrant
        returns (
            uint256 shares,
            uint256 baseInput,
            uint256 quoteInput
        )
    {
        // The balance of baseToken and quoteToken should be the balance minus the fee
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this)) - _MT_FEE_BASE_;
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this)) - _MT_FEE_QUOTE_;
        // The reserve of baseToken and quoteToken
        uint256 baseReserve = _BASE_RESERVE_;
        uint256 quoteReserve = _QUOTE_RESERVE_;

        // The amount of baseToken and quoteToken user transfer to GSP
        baseInput = baseBalance - baseReserve;
        quoteInput = quoteBalance - quoteReserve;

        // BaseToken should be transferred to GSP before calling buyShares
        require(baseInput > 0, "NO_BASE_INPUT");

        // Round down when withdrawing. Therefore, never be a situation occuring balance is 0 but totalsupply is not 0
        // But May Happen，reserve >0 But totalSupply = 0
        if (totalSupply == 0) { 
            // case 1. initial supply
            // The shares will be minted to user
            shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_) 
                ? DecimalMath.divFloor(quoteBalance, _I_)
                : baseBalance;
            // The target will be updated
            _BASE_TARGET_ = uint112(shares);
            _QUOTE_TARGET_ = uint112(DecimalMath.mulFloor(shares, _I_));
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
            uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
            uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
            // The shares will be minted to user
            shares = DecimalMath.mulFloor(totalSupply, mintRatio);

            // The target will be updated
            _BASE_TARGET_ = uint112(uint256(_BASE_TARGET_) + (DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)));
            _QUOTE_TARGET_ = uint112(uint256(_QUOTE_TARGET_) + (DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)));
        }
        // The shares will be minted to user
        // The reserve will be updated
        _mint(to, shares); //@note no checks for address0
        _setReserve(baseBalance, quoteBalance);
        emit BuyShares(to, shares, _SHARES_[to]);
    }
```
so does the `_mint` function in the GSPVault, which mints shares to the users.

```solidity
    function _mint(address user, uint256 value) internal {
        require(value > 1000, "MINT_AMOUNT_NOT_ENOUGH");
        _SHARES_[user] = _SHARES_[user] + value;
        totalSupply = totalSupply + value;
        emit Mint(user, value);
        emit Transfer(address(0), user, value);
    }
```

## Impact
Minting to a zero address causes shares to be lost. Note also that, the totalSupply is also increased, causing the shares in circulation differs from actual totalSupply of shares, this results in a host of accounting issues. It also affects the amount of tokens transferred to other users who decide to sell their shares.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L31
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L294

## Tool used

Manual Review

## Recommendation
Consider adding a 0 address check in the `_mint` and `buyShares` functions.