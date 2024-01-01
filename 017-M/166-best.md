Kind Indigo Kookaburra

high

# `DODOMath._SolveQuadraticFunctionForTrade` is implemented incorrect

## Summary
`DODOMath` is building block of all the computations. It computed complex mathematics operations like integration and Quadratic equations. 
`DODOMath._SolveQuadraticFunctionForTarget` calculates the root of quadratic equation based on given params but using incorrect formula and so creating an array of issues

## Vulnerability Detail
From the comments and docs 
```solidity
        Follow the integration expression above, we have:
        i*deltaB = (Q2-Q1)*(1-k+kQ0^2/Q1/Q2)
        Given Q1 and deltaB, solve Q2
        This is a quadratic function and the standard version is
        aQ2^2 + bQ2 + c = 0, where
        a=1-k
        -b=(1-k)Q1-kQ0^2/Q1+i*deltaB
        c=-kQ0^2 
        and Q2=(-b+sqrt(b^2+4(1-k)kQ0^2))/2(1-k)
        note: another root is negative, abondan
```
As mentioned above ` -b=(1-k)Q1-kQ0^2/Q1+i*deltaB `  which is used to compute discriminant of the quadratic. 

```solidity
116   function _SolveQuadraticFunctionForTrade(
117        uint256 V0,
118       uint256 V1,
119        uint256 delta,
120        uint256 i,
121        uint256 k
122    ) internal pure returns (uint256) {
  ....................................................................
133        if (k == DecimalMath.ONE) {
134            // if k==1
135            // Q2=Q1/(1+ideltaBQ1/Q0/Q0)
136            // temp = ideltaBQ1/Q0/Q0
137            // Q2 = Q1/(1+temp)
138            // Q1-Q2 = Q1*(1-1/(1+temp)) = Q1*(temp/(1+temp))
139            // uint256 temp = i.mul(delta).mul(V1).div(V0.mul(V0));
140            uint256 temp;
141            uint256 idelta = i * (delta);
142            if (idelta == 0) {
143                temp = 0;
144            } else if ((idelta * V1) / idelta == V1) {
145                temp = (idelta * V1) / (V0 * V0);
146            } else {
147                temp = delta * (V1) / (V0) * (i) / (V0);
148            }
149            return V1 * (temp) / (temp + (DecimalMath.ONE));
 150       }

152       // calculate -b value and sig
153      // b = kQ0^2/Q1-i*deltaB-(1-k)Q1
154       // part1 = (1-k)Q1 >=0
155        // part2 = kQ0^2/Q1-i*deltaB >=0
156        // bAbs = abs(part1-part2)
157        // if part1>part2 => b is negative => bSig is false
158       // if part2>part1 => b is positive => bSig is true
159        uint256 part2 = k * (V0) / (V1) * (V0) + (i * (delta)); // kQ0^2/Q1-i*deltaB
160       uint256 bAbs = (DecimalMath.ONE - k) * (V1); // (1-k)Q1

162        bool bSig;
        if (bAbs >= part2) {
            bAbs = bAbs - part2;
            bSig = false;
        } else {
            bAbs = part2 - bAbs;
            bSig = true;
        }
        bAbs = bAbs / (DecimalMath.ONE);

...........................................................................................
}
```
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L116C2-L170C41
```solidity
       as we only support sell amount as delta, the deltaB is always negative
        the input ideltaB is actually -ideltaB in the equation
```
Also, from the comments it has been stated `ideltaB is actually -ideltaB in the equation` we need to modify the above computation terms in the same way
But in same function at two instance delta has been used differently which is incorrect 

First one is in the calculation of `temp=ideltaBQ1/Q0/Q0` at Line 149 where deltaB is used as it is 
and 
Second at Line 159 where sign has been used differently


## Impact
`DODOMath._SolveQuadraticFunctionForTarget` is used all over the place and Incorrect output will break all the trades/PMM.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L116C2-L170C41
## Tool used

Manual Review

## Recommendation
First Write better comments and docs for Library since it's too confusing.
A good mitigation will be taking delta input as int and use original formula to compute roots of the Quadratic equation 