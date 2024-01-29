Pin Lock Contract
=================================

* Author: Robert Kornacki
* Created: October 2 2020
* License: CC0
* Difficulty: Beginner
* Ergo Playground Link: [Pin Lock Contract](https://scastie.scala-lang.org/FBQnxS8iQBqrt8MM2dykLw)

Description
----------
This is a simple smart contract which locks deposited funds under a pin number. When the pin number is initially provided by the user, it is hashed off-chain, and the hash itself is stored on-chain.

When the user wishes to withdraw his/her funds(spend the utxo locked under this contract) he/she submits a `withdrawTransaction` with the actual pin number in a register of the output utxo. The contract then executes and checks that when the newly provided pin number gets hashed, it is equal to the original hash held in the register of the box locked by the contract. If they are equal then the contract validates, and the transaction is accepted as being valid and added to the blockchain history.

Do note, this contract is purely intended to be used as an educational example. It is completely insecure and should never be used on-chain. Bad actors can scan the mempool to find the pin number when the `withdrawTransaction` is posted, and then front-run by posting a tx with a higher tx fee + the same pin number, thereby stealing your locked funds. That said, this is a great simple contract to wet your feet with.



Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/FBQnxS8iQBqrt8MM2dykLw)

```scala

import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
import org.ergoplatform.Pay2SAddress
import sigmastate.eval.Extensions._
import scorex.crypto.hash.{Blake2b256}

///////////////////////////////////////////////////////////////////////////////////
// Create Pin Lock Contract //
///////////////////////////////////////////////////////////////////////////////////
// Create a Pin Lock script which requires the user to submit a PIN number
// Pin number initially is hashed before being stored on-chain in R4.
// To withdraw user must submit the Pin number which gets posted
// in R4 of the output box as cleartext and hashed to check against
// the stored hash in the input box R4.
val pinLockScript = s"""
  sigmaProp(SELF.R4[Coll[Byte]].get == blake2b256(OUTPUTS(0).R4[Coll[Byte]].get))
""".stripMargin
val pinLockContract = ErgoScriptCompiler.compile(Map(), pinLockScript)

// Build the P2S Address of the contract.
// This is not needed for the code at hand, but is demonstrated here as a reference
// to see how to acquire the P2S address so you can use contracts live on mainnet.
val contractAddress = Pay2SAddress(pinLockContract.ergoTree)
println("Pin Lock Contract Address: " + contractAddress)
println("-----------")



///////////////////////////////////////////////////////////////////////////////////
// Prepare A Test Scenario //
///////////////////////////////////////////////////////////////////////////////////
// Create a simulated blockchain (aka "Mockchain")
val blockchainSim = newBlockChainSimulationScenario("PinLock Scenario")
// Define an actor/user (with a wallet tied to said Party)
val userParty = blockchainSim.newParty("buyer")
// Define example user input
val pinNumber = "1293"
// Hash the pin number
val hashedPin = Blake2b256(pinNumber.getBytes())
// Define initial nanoErgs in the user's wallet
val userFunds = 100000000
// Generate initial userFunds in the user's wallet
userParty.generateUnspentBoxes(toSpend = userFunds)
userParty.printUnspentAssets()
println("-----------")



///////////////////////////////////////////////////////////////////////////////////
// Deposit Funds Into Pin Lock Contract //
///////////////////////////////////////////////////////////////////////////////////
// Create an output box with the user's funds locked under the contract
val pinLockBox      = Box(value = userFunds/2,
                          script = pinLockContract,
                          register = (R4 -> hashedPin))
// Create the deposit transaction which locks the users funds under the contract
val depositTransaction = Transaction(
      inputs       = userParty.selectUnspentBoxes(toSpend = userFunds),
      outputs      = List(pinLockBox),
      fee          = MinTxFee,
      sendChangeTo = userParty.wallet.getAddress
    )

// Print depositTransaction
println(depositTransaction)
// Sign the depositTransaction
val depositTransactionSigned = userParty.wallet.sign(depositTransaction)
// Submit the tx to the simulated blockchain
blockchainSim.send(depositTransactionSigned)
userParty.printUnspentAssets()
println("-----------")


///////////////////////////////////////////////////////////////////////////////////
// Withdraw Funds Locked Under Pin Lock Contract //
///////////////////////////////////////////////////////////////////////////////////
// Create an output box which withdraws the funds to the user
// Subtracts `MinTxFee` from value to account for tx fee which
// must be paid.
val withdrawBox      = Box(value = userFunds/2 - MinTxFee,
                          script = contract(userParty.wallet.getAddress.pubKey),
                          register = (R4 -> pinNumber.getBytes()))

// Create the withdrawTransaction
val withdrawTransaction = Transaction(
      inputs       = List(depositTransactionSigned.outputs(0)),
      outputs      = List(withdrawBox),
      fee          = MinTxFee,
      sendChangeTo = userParty.wallet.getAddress
    )
// Print withdrawTransaction
println(withdrawTransaction)
// Sign the withdrawTransaction
val withdrawTransactionSigned = userParty.wallet.sign(withdrawTransaction)
// Submit the withdrawTransaction
blockchainSim.send(withdrawTransactionSigned)

// Print the user's wallet, which shows that the coins have been withdrawn (with same total as initial, minus the MinTxFee * 2)
userParty.printUnspentAssets()
println("-----------")
```
