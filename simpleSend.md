Simple Send
=================================

* Author: Crypto Cream
* Created: September 13 2021
* License: CC0
* Difficulty: Starter
* Ergo Playground Link: [Simple Send](https://scastie.scala-lang.org/Hb8INmnpQkOSP4MwL6eVNw)

Description
----------
A simple starting contract to help you understand the basics of the UTxO model and the Ergo Protocol.

1. Creates box for depositing funds from a sender party
2. The sender party signs transaction so funds are sent and locked into the a box under the smart contract, which is just a simple true statement for demonstration purpose
3. Another box is created to withdraw funds
4. This time the script being used is the receiver party's public key, therefore giving the receiver party permission to spend the unspent box
5. The transaction is signed by the sender, allowing the unspent box to be created, effectivley moving funds from the sender party to the receiver party

##### Key Concepts

- Ergo is a UTXO based blockchain with Proof-of-Work consensus
- Ergo is considered an *extended-UTXO model*, supporting advanced financial contracts similar to those in Ethereum's account-based model
- Since Ergo is UTXO based, therefore ErgoScript has many UTXO-specific constructs such as: 
  - `Box`, `INPUTS`, `OUTPUTS`
  - A complete list is available [here](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/docs/LangSpec.md)
- A `Box` is essentially a UTXO and consists of up to ten registers for storing data. Similar to Bitcoin, a transaction spends one or more existing boxes (denoted using the `INPUTS` array), and creates one or more new boxes (denoted using the `OUTPUTS` array)
- ErgoScript's syntax is a subset of Scala's. However, knowledge of Scala is not necessary to learn ErgoScript because the amount of Scala needed to write ErgoScript is small e.g. `val`
- Note that arrays in Scala are accessed using round parentheses, not square brackets like in Java or Python. Thus, `OUTPUTS(0)` refers to the first element of the `OUTPUTS` array
- Unlike Scala, ErgoScript does not support the `var` keyword, and thus everything is immutable

##### Anatomy of an Ergo transaction

An Ergo transaction consists of:

1. One or more **Input** boxes (source of funds). These boxes must already exist and will be destroyed. The guard script in each of these boxes will be evaluated and must return `true` for the transaction to be considered valid
2. One or more **Output** boxes (destination of funds), these boxes will be created

Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/Hb8INmnpQkOSP4MwL6eVNw)
```scala

// Don't worry about these imports, this is just so we can test our contract in the Ergo Playground
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._


// This is used to create a simulated blockchain (aka "Mockchain") for the Ergo Playground
val blockchainSim = newBlockChainSimulationScenario("Send Scenario")


// Here we define our users for the Ergo Playground scenario, and assigns the users a wallet
val senderParty = blockchainSim.newParty("sender")
val receiverParty = blockchainSim.newParty("receiver")


// Define initial nanoErgs in the users wallet (1 Erg = 1000000000 NanoErgs) 
// We will give the receiver 0 funds to demonstrate the change in balance
val senderFunds = 100000000
val receiverFunds = 0


// Generate initial userFunds in the users wallet, this is just to help us with the scenario by giving the sender unspent UTxO's
// We print the assets to easier demonstrate
senderParty.generateUnspentBoxes(toSpend = senderFunds)
senderParty.printUnspentAssets()
receiverParty.printUnspentAssets()
println("---------------------")


// This the actual ErgoScript, which gets compiled ready for use
// Here we will put a simple true statement for the transaction to be considered valid
// This is very dangerous, as it will create a box that gives anyone permission to spend, however we are just demonstrating
val trueScript = "sigmaProp(1 == 1)"
val trueContract = ErgoScriptCompiler.compile(Map(), trueScript)


// Creating our first Box
// - In Ergo, a transaction output (whether spent or unspent) is called a box
// - A box can have 10 Registers (R) that contain particular bits of data
// - A box at the minimum has 4 pieces of information
//   For now we will only worry about setting these 2:
//   1. The value in NanoErgs (1 Erg = 1000000000 NanoErgs) (R0)
//   2. The guard script, this is essentially the "smart contract" (R1)
//   These will be set automatically:
//   3. Additional assets (tokens) stored in this box (R2)
//   4. Creation info of the box (txId, the identifier of the transaction that created the box along with an output index) (R3)
// In addition, a box can have 6 optional registers (R4-R9) to store custom data for use in smart contracts

// Create an output box with the users funds to be locked under the true contract
val trueBox      = Box(value = 50000000,
                        script = trueContract)

// Create the deposit transaction
// Remember we need inputs and outputs
// We also need to set the fee which we set to the minimum, and any change will get sent back to the sender
val depositTransaction = Transaction(
      inputs       = senderParty.selectUnspentBoxes(toSpend = senderFunds),
      outputs      = List(trueBox),
      fee          = MinTxFee,
      sendChangeTo = senderParty.wallet.getAddress
    )


// Print deposit transaction for demo purposes
println(depositTransaction)


// Next we need to sign the deposit transaction, so we know the request came from the sender party
val depositTransactionSigned = senderParty.wallet.sign(depositTransaction)


// We then submit the tx to the mockchain and print so we can see the result
blockchainSim.send(depositTransactionSigned)
senderParty.printUnspentAssets()
println("---------------------")


// Next we will withdraw the funds from the True Contract 
// We will create an output box which withdraws the funds for the receiver
// We also need to subtract MinTxFee from value to account for tx fee we paid earlier (has to be paid)
// This time instead of using our compiled Ergo Script, we use the receiver partys public key as the guard script
// This essentially explicitly gives only the receiver party to spend the box, as only they have the corresponding private key
val withdrawBox      = Box(value = 50000000 - MinTxFee,
                          script = contract(receiverParty.wallet.getAddress.pubKey))

// In similar fashion, we create the withdraw transaction
// We set the input as the previous transaction (the 0 represents the first output, in an array
// in this case is the only output)
val withdrawTransaction = Transaction(
      inputs       = List(depositTransactionSigned.outputs(0)),
      outputs      = List(withdrawBox),
      fee          = MinTxFee,
      sendChangeTo = senderParty.wallet.getAddress
    )


// Print withdrawTransaction for demonstration purposes
println(withdrawTransaction)


// Again we sign the withdraw transaction
val withdrawTransactionSigned = senderParty.wallet.sign(withdrawTransaction)


// Submit the withdraw transaction to the mockchain
blockchainSim.send(withdrawTransactionSigned)


// Print the users wallets, which shows the nanoErg have been sent (with same overall total as the initial amount
// minus the MinTxFee * 2)
senderParty.printUnspentAssets()
receiverParty.printUnspentAssets()
println("---------------------")

```