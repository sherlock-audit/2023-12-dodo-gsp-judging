Ancient Raspberry Wallaby

high

# `GSPFunding::buyShares()` doesn't refund unused tokens according to the `mintRatio`

## Summary
Shares are minted to users based on the `input amount / reserve amount` ratio. Both `baseInputRatio` and `quoteInputRatio` are calculated, and the smaller one is used to determine how many shares will be minted. However, the excess amount of tokens ( the one with the bigger ratio) are never refunded back to the user. It is impossible for a user to send tokens with the exact same ratios all the time since reserves are constantly changing. The protocol must refund excess tokens to users.

## Vulnerability Detail
Users transfer tokens to `GSP`, and then call the `buyShares` function. This function checks how many tokens are transferred, and then calculates input ratios of these tokens compared to the current reserves.

The amount of shares to mint to the user is determined based on these input ratios, and the smaller ratio is used while minting. However, the excess amount of tokens (*some portion of the token with bigger input ratio*) are never refunded back to the user.  
[https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L67C1-L75C124](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L67C1-L75C124)

```solidity
    function buyShares(address to)
        ...
    {
        ...

        // The amount of baseToken and quoteToken user transfer to GSP
        baseInput = baseBalance - baseReserve;
        quoteInput = quoteBalance - quoteReserve;
        
        // ... some code
         
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
            uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
-->         uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
            // The shares will be minted to user
-->         shares = DecimalMath.mulFloor(totalSupply, mintRatio); //@audit-issue User can not know the exact ratios. Reserves constantly change. One token will be sent more than other but the share is calculated based on the smaller one. Excess token amount should be transferred back to the user

-->         // The target will be updated //@audit both targets are also updated with the mintRatio, but reserves are updated without refunding excess amount back -> Which means there will be a mismatch between target and reserve. 
            _BASE_TARGET_ = uint112(uint256(_BASE_TARGET_) + (DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)));
            _QUOTE_TARGET_ = uint112(uint256(_QUOTE_TARGET_) + (DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)));
        }
        // The shares will be minted to user
        // The reserve will be updated
        _mint(to, shares);
        _setReserve(baseBalance, quoteBalance);
        emit BuyShares(to, shares, _SHARES_[to]);
    }
```

As we can see above, the smaller input ratio is used to determine `shares` amount and there is no refund mechanism for the excess amount. Users will lose some of their tokens.

A user can not know the exact ratios all the time and one token will always sent more than the other. Even if a user transfers perfect amount of tokens by calculating the current ratios before sending the transaction, the reserves may change just before that transaction is executed.

Moreover, there is also another impact. As we can see [here](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L74C3-L75C124), both base and quote targets are updated based on the `mintRatio`. However, the reserves are [updated](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L80) without refunding tokens back. This will create a mismatch between target and reserve, which will cause additional problems like incorrect R ratios etc.

## Impact

- Loss of funds for the user.
- Mismatch between reserves and target values.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L67C1-L80C48

## Tool used

Manual Review

## Recommendation
Unused tokens should be refunded to the user. 
The amount of unused tokens are (_assuming `baseInputRatio` is the smaller one_): `(quoteInputRatio - mintInputRatio) * quoteInput`