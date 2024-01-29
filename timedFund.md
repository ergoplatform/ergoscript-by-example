Timed Fund
=================================

- Author: Catalyst244
- Created: September 27 2021
- License: CC0
- Difficulty: Beginner
- Ergo Playground Link: [Timed Fund Contract](https://scastie.scala-lang.org/AI09G8XrRk2t3pne504zuA)

Description
-----------

A simple starting contract to help you to have a better understanding on blockchain height and mixing sigma-statements with other statements in Ergo.

In this example first party (Alice) is going to create a deposit which is going to be available to second party (Bob) until a certail time 
after that Alice has the ability to take the fund back and Bob doesn't have access anymore.

This is a very simple and plain contract to show the usage of height and mixing sigma-statements with other statements 
but using this contract you can create more complex contracts like crowdfunding.

Code
----------

#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/AI09G8XrRk2t3pne504zuA)

```scala
// Timed fund 
// Don't worry about these imports, this is just so we can test our contract in the Ergo Playground
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._


// This is used to create a simulated blockchain (aka "Mockchain") for the Ergo Playground
val blockchainSim = newBlockChainSimulationScenario("Timed Fund Scenario")

// In ergo contracts we work with neno ergs, each erg equals 1000000000 nanoergs 
val nanoergsInErg = 1000000000L 

// Define first party who is going to make the timed fund called Alice
val alice = blockchainSim.newParty("Alice")
val aliceFunds = 200*nanoergsInErg

// Define second party who is going to have access to funds untill a certain time called Bob
val bob = blockchainSim.newParty("Bob")

// Define deposit amount
val depositAmount = 100*nanoergsInErg

// Generate initial userFunds in the users wallet, this is just to help us with the scenario by giving the sender unspent UTxO's
// We print the assets to easier demonstrate
alice.generateUnspentBoxes(toSpend = aliceFunds)
alice.printUnspentAssets()
bob.printUnspentAssets()
println("---------------------")

///////////////////////////////////////////
// Timed Fund Contract                   //
///////////////////////////////////////////

// In this example Alice is going to create a deposit which is going to be available to Bob until a certail time. 
// After that Alice has the ability to take the fund back.
// This is a very simple and plain contract to show the usage of height and mixing sigma-statements with other statements.
// But using this contract you can create more complex contracts like crawdfunding.

val timedFundScript = s""" alicePK && sigmaProp(HEIGHT > deadline) || bobPK && sigmaProp(HEIGHT <= deadline) """.stripMargin

// In a real world contract you should set the deadline based on the current hight of the chain which is the sequential block number of the block in which the script is evaluated.
// You can see the average time it takes to add a block to the chain in explorer stats: https://explorer.ergoplatform.com/en/stats
// Also to get the height of the chain you can use mainnet, testnet or localnet API based on where you're testing your code 
// Here's a link to mainnet API: https://api.ergoplatform.com/api/v1/docs/
val deadline = blockchainSim.getHeight + 500

// Compile the contract with an included `Map` which specifies what the values of given parameters are going to be hard-coded into the contract
val timedFundContract = ErgoScriptCompiler.compile(Map("deadline" -> deadline, 
                                                       "bobPK" -> bob.wallet.getAddress.pubKey,
                                                       "alicePK" -> alice.wallet.getAddress.pubKey
                                                      ), timedFundScript)

// As you know in Ergo, a transaction output (whether spent or unspent) is called a box
// Create an output box with the users funds to be locked under the timedFund contract
val timedFundBox      = Box(value = depositAmount, 
                           script = timedFundContract)

// Create the deposit transaction
// Remember we need inputs and outputs
// We also need to set the fee which we set to the minimum, and any change will get sent back to the sender (Alice)
val timedFundTransaction = Transaction(
      inputs       = alice.selectUnspentBoxes(toSpend = aliceFunds),
      outputs      = List(timedFundBox),
      fee          = MinTxFee,
      sendChangeTo = alice.wallet.getAddress
    )

// Print transaction for demo purposes
println(timedFundTransaction)
println("---------------------")

// Next we need to sign the deposit transaction, so we know the request came from alice
val timedFundTransactionSigned = alice.wallet.sign(timedFundTransaction)


// We then submit the transaction to the mockchain and print so we can see the result
// As you see Alice funds are reduced by 100 ergs plus what ever the minimun transaction fee was
blockchainSim.send(timedFundTransactionSigned)
alice.printUnspentAssets()
println("---------------------")

// We have two scenarios here: 
//  - Bob withdraws the fund before deadline 
//  - Alice withdraw the fund after deadline
// We flip a random coin to choose the scenario. (Resave to run the simulation again)
val bobWithdraws = scala.util.Random.nextBoolean

if (bobWithdraws) {
  
  println("===== Bob withdraws the funds before deadline =====")

  // We will create an output box which withdraws the funds for Bob
  // We also need to subtract MinTxFee from value to account for tx fee we paid earlier (has to be paid)
  // This time instead of using our compiled Ergo Script, we use the Bob's public key as the guard script
  // This essentially explicitly gives only the receiver party to spend the box, as only they have the corresponding private key
  val bobWithdrawBox      = Box(value = depositAmount - MinTxFee,
                            script = contract(bob.wallet.getAddress.pubKey))

  // Now we create the withdraw transaction
  // We set the input as the previous transaction (the 0 represents the first output, in an array
  // in this case is the only output)
  val bobWithdrawTransaction = Transaction(
        inputs       = List(timedFundTransactionSigned.outputs(0)),
        outputs      = List(bobWithdrawBox),
        fee          = MinTxFee,
        sendChangeTo = bob.wallet.getAddress
      )

  // Again we sign the withdraw transaction
  val bobWithdrawTransactionSigned = bob.wallet.sign(bobWithdrawTransaction)


  // Submit the withdraw transaction to the mockchain
  blockchainSim.send(bobWithdrawTransactionSigned)
}
else {
  println("===== Alice withdraws the funds after deadline =====")
  
  // In a real world situation we should wait until height reaches the deadline that alice has set but here in mockchain we can just manipulate chain height
  blockchainSim.setHeight(deadline + 1)
  
  // The rest is similar to what we did in past scenario 
  
  // We will create an output box which withdraws the funds for Alice
  // We also need to subtract MinTxFee from value to account for tx fee we paid earlier (has to be paid)
  // This time instead of using our compiled Ergo Script, we use the Alice's public key as the guard script
  // This essentially explicitly gives only the receiver party to spend the box, as only they have the corresponding private key
  val aliceWithdrawBox      = Box(value = depositAmount - MinTxFee,
                            script = contract(alice.wallet.getAddress.pubKey))

  // Now we create the withdraw transaction
  // We set the input as the previous transaction (the 0 represents the first output, in an array
  // in this case is the only output)
  val aliceWithdrawTransaction = Transaction(
        inputs       = List(timedFundTransactionSigned.outputs(0)),
        outputs      = List(aliceWithdrawBox),
        fee          = MinTxFee,
        sendChangeTo = alice.wallet.getAddress
      )

  // Again we sign the withdraw transaction
  val aliceWithdrawTransactionSigned = alice.wallet.sign(aliceWithdrawTransaction)


  // Submit the withdraw transaction to the mockchain
  blockchainSim.send(aliceWithdrawTransactionSigned)
}


// Print the users wallets, which shows the nanoErg have been sent (with same overall total as the initial amount minus the MinTxFee * 2)
alice.printUnspentAssets()
bob.printUnspentAssets()
println("---------------------")
```
