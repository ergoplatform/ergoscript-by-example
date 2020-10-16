# ErgoScript By Example
Learn ErgoScript by reading example smart contracts powered by the Ergo Playground.

Each contract example includes a `Ergo Playground` link which allows you to instantly edit and run the smart contract code inside of your browser.

If you ever need clarity about how specific types/functions/operators in ErgoScript work, please reference the [ErgoScript Language Description](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/docs/LangSpec.md).

For an overarching summary of how everything in this repo works, please reference the video below:
#### [ErgoScript By Example & Ergo Playground Introductory Video](https://www.youtube.com/watch?v=8l2v1asHgyA)


## Table Of Contents
- [ErgoScript Examples](<#Ergoscript-examples>)
- [How To Contribute](<#how-to-contribute>)


## ErgoScript Examples

| Number | Difficulty | Title |
| ---  | ---  | ---  |
| 1 | Beginner | [Pin Lock Contract](pinLockContract.md) |
| 2 | Intermediate | [Single-Chain Swap Contracts](singleChainSwap.md) |


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
