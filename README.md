# 🧨 Damn Vulnerable DeFi — Solutions (Foundry Edition)

---

## 🏗️ Overview

This repository contains my personal implementations and walkthroughs for each of the **Damn Vulnerable DeFi** (DVD) challenges.  
Each challenge folder includes:

- 🧠 Step-by-step reasoning and vulnerability analysis
- 🧪 Foundry test files
- ⚙️ Deployment & test scripts

My goal was not only to solve the levels, but also to understand **how and why** each vulnerability occurs — covering reentrancy, flash loans, signature replay, oracle manipulation, delegatecall abuse, and more.

---

## 🧩 What to do before 

#### 1. First, we need to clone the repo to our machine:

```bash
git clone https://github.com/theredguild/damn-vulnerable-defi.git
cd damn-vulnerable-defi-foundry
```

### 2. Make sure `foundry` is installed. If not, followe these steps:

```bash
forge install
forge build
forge test -vv
```

## 🧠 Topics Covered

| Category                 | Key Concepts                           |
| ------------------------ | -------------------------------------- |
| **Flash Loans**          | Instant liquidity abuse, fee bypassing |
| **Reentrancy**           | Draining vulnerable pools              |
| **Access Control**       | Misconfigured admin roles              |
| **Delegatecall**         | Logic hijacking via proxy              |
| **Oracle Manipulation**  | Price feed tampering                   |
| **Signature Replay**     | Reusing valid off-chain signatures     |
| **ERC20/ERC777 Quirks**  | Callback exploitation                  |
| **Upgradeability Flaws** | Storage layout corruption              |




## ⚠️ Disclaimer

These solutions are for educational and research purposes only.
Please use this knowledge ethically — the intent is to learn security principles, not exploit real systems.

## ⭐ Contribute

If you’ve improved a test, found a cleaner exploit, or want to add more commentary, feel free to fork and open a PR.
Knowledge grows when shared.

