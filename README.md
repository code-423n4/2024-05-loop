
# LoopFi audit details
- Total Prize Pool: $12,100 in USDC
  - HM awards: $8,640 in USDC
  - QA awards: $360 in USDC 
  - Judge awards: $1,600 in USDC
  - Validator awards: $1,000 in USDC 
  - Scout awards: $500 in USDC
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/audits/2024-05-loopfi/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts May 1, 2024 20:00 UTC
- Ends May 8, 2024 20:00 UTC

## Automated Findings / Publicly Known Issues

The 4naly3er report can be found [here](https://github.com/code-423n4/2024-05-loop/blob/main/4naly3er-report.md).



_Note for C4 wardens: Anything included in this `Automated Findings / Publicly Known Issues` section is considered a publicly known issue and is ineligible for awards._

- Crafting malicious calldata so that users get less funds as expected when claiming lpETH on LRT deposits (e.g. by setting big slippage/price impact)
- Owner setting malicious lpETH and lpETHVault contracts. Users have 7 days to withdraw in that case.
- Previous audit findings are out of scope.


# Overview

# LoopFi Prelaunch Point Contracts

![](https://github.com/code-423n4/2024-05-loop/blob/main/img/icon-loop.png?raw=true)

## Description

Users can lock ETH, WETH and wrapped LRTs into this contract, which will emit events tracked on a backed to calculate their corresponding amount of points. When staking, users can use a referral code encoded as `bytes32` that will give the referral extra points.

When Loop contracts are launched, the owner of the contract can call only once `setLoopAddresses` to set the `lpETH` contract as well as the staking vault for this token. This activation date is stored at `loopActivation`.

Once these addresses are set, all deposits are paused and users have `7 days` to withdraw their tokens in case they changed their mind, or they detected a malicious contract being set. On withdrawal, users loose all their points.

After these `7 days` the owner can call `convertAllETH`, that converts all ETH in the contract for `lpETH`. This conversion has the timestamp `startClaimDate`. The conversion for LRTs happens on each claim by using 0x API. This is triggered by each user.

After the global ETH conversion, users can start claiming their `lpETH` or claiming and staking them in a vault for extra rewards. The amount of `lpETH` they receive is proportional to their locked ETH amount or the amount given by the conversion by 0x API. The minimum amount to receive is determined offchain and controlled by a slippage parameter in the frontend dApp.

### Notes:

- On deployment the variable `loopActivation` is set to be 120 days into the future. If owner does not set the Loop contracts before this date, the contract becomes unusable except for users to withdraw their ETH and other locked tokens from this contract.
- There is an emergency mode that allows users to withdraw without any time restriction. If ETH was converted already users can call `claim` instead. This mode ensures that LRTs are not locked in the contract in case 0x stops working as intended.

## Links

- **Previous audits:**  [First Audit Report](https://notes.watchpug.com/p/18f09102eb9ghBQP) - 
[Second Audit Report](https://notes.watchpug.com/p/18f13f46b9eE_mgb)
- **Documentation:** https://docs.loopfi.xyz/
- **Website:** https://www.loopfi.xyz/
- **X/Twitter:** https://twitter.com/loopfixyz
- **Discord:** https://discord.gg/r25fukGw

---

# Scope
See [scope.txt](https://github.com/code-423n4/2024-05-loop/blob/main/scope.txt)
### Files in scope


| File   | Logic Contracts | Interfaces | SLOC  | Purpose | Libraries used |
| ------ | --------------- | ---------- | ----- | -----   | ------------ |
| [/src/PrelaunchPoints.sol](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol) | 1| **** | 296 | |@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol<br>@openzeppelin/contracts/utils/math/Math.sol|
| **Totals** | **1** | **** | **296** | | |

### Files out of scope

*See [out_of_scope.txt](https://github.com/code-423n4/2024-05-loop/blob/main/out_of_scope.txt)*

| File         |
| ------------ |
| ./script/PrelaunchPoints.s.sol |
| ./src/interfaces/ILpETH.sol |
| ./src/interfaces/ILpETHVault.sol |
| ./src/interfaces/IWETH.sol |
| ./src/mock/AttackContract.sol |
| ./src/mock/MockERC20.sol |
| ./src/mock/MockLRT.sol |
| ./src/mock/MockLpETH.sol |
| ./src/mock/MockLpETHVault.sol |
| ./test/PrelaunchPoints.t.sol |
| Totals: 10 |

## Scoping Q &amp; A

### General questions


| Question                                | Answer                       |
| --------------------------------------- | ---------------------------- |
| ERC20 used by the protocol              |    WETH, weETH, ezETH, rsETH, rswETH, uniETH, pufETH            |
| Test coverage                           | 78.95%                          |
| ERC721 used  by the protocol            |            None              |
| ERC777 used by the protocol             |           None                |
| ERC1155 used by the protocol            |              None            |
| Chains the protocol will be deployed on | Ethereum |


### External integrations (e.g., Uniswap) behavior in scope:


| Question                                                  | Answer |
| --------------------------------------------------------- | ------ |
| Enabling/disabling fees (e.g. Blur disables/enables fees) | No   |
| Pausability (e.g. Uniswap pool gets paused)               |  No   |
| Upgradeability (e.g. Uniswap gets upgraded)               |   No  |


### EIP compliance checklist
None


# Additional context

## Main invariants

- Only the owner can set new accepted LRTs, change mode to emergency mode on failure of 0x integration, and set a new owner
- Deposits are active up to the lpETH contract and lpETHVault contract are set
- Withdrawals are only active on emergency mode or during 7 days after loopActivation is set
- Users that deposit ETH/WETH get the correct amount of lpETH on claim (1 to 1 conversion)
- Users that deposit LRTs get the correct amount assuming a favorable swap to ETH


## Attack ideas (where to focus for bugs)
- Malicious 0x protocol calldata crafting by users to steal funds on claim
- User funds getting locked forever

## All trusted roles in the protocol


| Role                                | Description                       |
| --------------------------------------- | ---------------------------- |
| Owner                          | Has access to privileged functions, contract owner             |


## Describe any novel or unique curve logic or mathematical models implemented in the contracts:

None


## Running tests


```bash
git clone https://github.com/code-423n4/2024-05-loop
git submodule update --init --recursive
cd 2024-05-loop
forge build
forge test
```

To run code coverage
```bash
forge coverage
```
To run gas benchmarks
```bash
forge test --gas-report
```

## Deployment
First, set your environment variables in a `.env` file as in `.env.example`. To load these variables run

```
source .env
```

To run the integration tests with 0x API, first create and fill a `.env` file with the keys of `.env.example`. Then, run

```
yarn hardhat test
```


To deploy and verify the `PrelaunchPoints` contract run

```
forge script script/PrelaunchPoints.s.sol --rpc-url $RPC_URL --broadcast --verify
```

## Audit metrics

To get some metrics pre-audit, run

```
yarn solidity-code-metrics src/PrelaunchPoints.sol > metrics.md
```
![Screenshot from 2024-04-30 17-39-45](https://github.com/code-423n4/2024-05-loop/blob/main/screenshot_1.png?raw=true)
![Screenshot from 2024-04-30 17-40-08](https://github.com/code-423n4/2024-05-loop/blob/main/screenshot_2.png?raw=true)
![Screenshot from 2024-04-30 17-40-23](https://github.com/code-423n4/2024-05-loop/blob/main/screenshot_3.png?raw=true)
![Screenshot from 2024-04-30 17-38-53](https://github.com/code-423n4/2024-05-loop/blob/main/screenshot_4.png?raw=true)

## Miscellaneous
Employees of LoopFi and employees' family members are ineligible to participate in this audit.
