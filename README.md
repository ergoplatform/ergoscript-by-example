# ErgoScript By Example
Learn ErgoScript by reading example smart contracts powered by the Ergo Playground.

Each contract example includes a `Ergo Playground` link which allows you to instantly edit and run the smart contract code inside of your browser.

If you ever need clarity about how specific types/functions/operators in ErgoScript work, please reference the [ErgoScript Language Description](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/docs/LangSpec.md).

For an overarching summary of how everything in this repo works, please reference the video below:
#### [ErgoScript By Example & Ergo Playground Introductory Video](https://www.youtube.com/watch?v=8l2v1asHgyA)


## Table Of Contents
- [ErgoScript Examples](<#Ergoscript-examples>)
- [Extra Resources To Get Started](<#Extra-Resources-To-Get-Started>)
- [Other Resources](<#other-resources>)
- [How To Contribute](<#how-to-contribute>)


## ErgoScript Examples

| Number | Difficulty | Title |
| ---  | ---  | ---  |
| 1 | Beginner | [Pin Lock Contract](pinLockContract.md) |
| 2 | Intermediate | [Single-Chain Swap Contracts](singleChainSwap.md) |
| 3 | Starter | [Simple Send](simpleSend.md) |
| 4 | Intermediate | [Double-Chain Swap Contracts](doubleChainSwap.md) |
| 5 | Beginner | [Timed Fund Contract](timedFund.md) |
| 6 | Beginner | [Grantor/Beneficiary Pin Lock Contract](grantorBeneficiaryPinLock.md) |
| 7 | Beginner | [Escrow Deposit Contract](escrowDepositContract.md) |
| 8 | Expert | [Token sales service contract](tokenSalesService.md) |
| 9 | Beginner | [Self-Replicating Sale Contract](selfReplicatingTokenSale.md) |
| 10 | Intermediate | [Heads or Tails game Contract](headsOrTails.md) |
| 11 | Expert| [Stealth Address](stealthAddress.md) |
| 12 | Expert | [Heads or Tails game Contract with Parallelization](headsOrTailsParallel.md) |

## Extra Resources To Get Started
If you are unfamiliar with the Extended UTXO model, smart contracts, or Ergo specifically, the above examples may be a little bit challenging to jump straight into. As such the following links below are recommended resources for getting a solid background in understanding what is going on:

1. [Emurgo Research: How Do UTXO Contracts Work?](https://github.com/Emurgo/Emurgo-Research/blob/master/smart-contracts/Unlocking%20The%20Potential%20Of%20The%20UTXO%20Model.md
) (First 2 sections)
2. [ErgoScript Whitepaper](https://ergoplatform.org/docs/ErgoScript.pdf)

3. [Advanced ErgoScript Whitepaper](https://ergoplatform.org/docs/AdvancedErgoScriptTutorial.pdf)

4. [Emurgo Research: High Level Design Patterns In EUTXO Systems](https://github.com/Emurgo/Emurgo-Research/blob/master/smart-contracts/High%20Level%20Design%20Patterns%20In%20Extended%20UTXO%20Systems.md)

The first two links should enlighten you to the majority of the basics, and the links thereafter are more advanced deep dives.

## Other Resources

- [Video: Ergoscript examples using AppKit](https://www.youtube.com/watch?v=Md5s-XV6-Hs) This video covers the lifecycle of a smart contract on ergo, how to interact with the blockchain. It also covers step by step some of the examples in this repository.

## How To Contribute

We are always happy to include new community-submitted examples within this repo if you are interested in contributing.

All examples submitted must:
- Use the `Ergo Playground` via Scastie ([basic template here](https://scastie.scala-lang.org/Uylafp7eQFyrIZdlvtdM0g))
- Create a test scenario to show off the off-chain logic required for the contract to work
- Provide clear comments inside of the code explaining what is going on
- Follow the [example template](example_template.md). (Ensure you include your scastie Ergo Playground url in both links)
- Have an understandable/relevant title + filename

Simply create a PR with your new `exampleTitle.md` file added to the repo + this `README.md` updated with your example added to the list of [ErgoScript Examples](<#Ergoscript-examples>).

If you are unsure about anything feel free to reference the [first example contract](pinLockContract.md) and/or join the [Ergo Discord](https://discord.gg/kj7s7nb) to ask questions.
