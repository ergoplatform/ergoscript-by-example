"Escrow deposit contract"
=================================

* Author: ThierryM
* Created: 2021-09-30
* License: CC0
* Difficulty: (Beginner)
* Ergo Playground Link: [Escrow deposit contract](https://scastie.scala-lang.org/ff7NAiXGRYKChAhsRDrMZA)

Description
----------
This contract is a simplified escrow deposit selling.
A neutral third party, called Validator here, will be implied in the sell from the Buyer to the Seller to resolve conflict after a defined period.

The Buyer will deposit funds in the contract and can at any time finalize the transaction to send the fund to the Seller.
The Seller can refund the buyer at any time.
The Validator can either force the payment or the refund after a defined period, here three weeks


Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/ff7NAiXGRYKChAhsRDrMZA)
```scala

// Escrow use case
// Required import for the  Ergo Playground
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._

// This is used to create a simulated blockchain (aka "Mockchain") for the Ergo Playground
val blockchainSim = newBlockChainSimulationScenario("Escrow use case")

// In ergo contracts we work with nano ergs, each erg equals 1000000000 nanoergs 
val nanoergsInErg = 1000000000L 

// Define the parties: Buyer, Seller and Validator
val buyer = blockchainSim.newParty("Buyer")
val seller = blockchainSim.newParty("Seller")
val validator = blockchainSim.newParty("Validator")

// Define deposit buyer funds and transaction price
val buyerFunds = 200*nanoergsInErg
val depositAmount = 100*nanoergsInErg

// Define the start of the litigation period in which the validator
// can finalize or cancel the purchase
// 720 block per day, starts after 3 weeks
val start_litigation = blockchainSim.getHeight + (720 * 7 * 3)
println("Starting the litigation period at height: " + start_litigation)

// Provide funds to the buyer
buyer.generateUnspentBoxes(toSpend = buyerFunds)

// Print the wallets before we start
buyer.printUnspentAssets()
seller.printUnspentAssets()
validator.printUnspentAssets()
println("---------------------")

///////////////////////////////////////////
// Escrow contract                       //
///////////////////////////////////////////

// The Buyer can finalize the payment to the seller at any time
// The Seller can refund the buyer at any time
// The Validator can force the payment to the seller or the refund to the buyer after 3 weeks
val escrowScript = s""" 
 (buyerPK && sigmaProp(OUTPUTS(0).propositionBytes == sellerPK.propBytes)) || (validatorPK && HEIGHT > start_litigation && (sigmaProp(OUTPUTS(0).propositionBytes == buyerPK.propBytes)|| sigmaProp(OUTPUTS(0).propositionBytes == sellerPK.propBytes)))|| (sellerPK && sigmaProp(OUTPUTS(0).propositionBytes == buyerPK.propBytes)) 
 """.stripMargin

// Compile the contract with an included `Map` which specifies what the values of given parameters are going to be hard-coded into the contract
val escrowContract = ErgoScriptCompiler.compile(Map("validatorPK" -> validator.wallet.getAddress.pubKey, 
                                                    "sellerPK" -> seller.wallet.getAddress.pubKey,
                                                    "buyerPK" -> buyer.wallet.getAddress.pubKey,
                                                    "start_litigation" -> start_litigation
                                                    ), escrowScript)

// Create an output box with the users funds to be locked until the validation or cancellation
val escrowBox = Box(value = depositAmount, 
                    script = escrowContract)

// Create the deposit transaction
// We also need to set the fee which we set to the minimum, and any change will get sent back to the buyer
val escrowTransaction = Transaction(
      inputs       = buyer.selectUnspentBoxes(toSpend = buyerFunds),
      outputs      = List(escrowBox),
      fee          = MinTxFee,
      sendChangeTo = buyer.wallet.getAddress
    )

// Print the deposit transaction
println(escrowTransaction)
println("---------------------")

// The Buyer signs the deposit transaction
val escrowTransactionSigned = buyer.wallet.sign(escrowTransaction)

// Submit the transaction to the mockchain and print buyer assets
blockchainSim.send(escrowTransactionSigned)
buyer.printUnspentAssets()
println("---------------------")

// We have now 4 scenario, run again to try another one:
// 1 - the buyer get the good after 1 week and process the finalization of the payment
// 2 - the buyer did not get the good after 4 weeks and ask the validator to proceed to a refund
// 3 - the seller did not get the fund after 4 weeks and ask the validator to proceed to the payment as the good was delivered
// 4 - the seller was not able to send the good after 2 week and refund the buyer

// Choose the scenario
val r = new scala.util.Random
val chooser = r.nextInt(4) + 1
println("===== Scenario " + chooser + " =====")

// In this scenario, the buyer get the good after 1 week and process the finalization of the payment
if (chooser == 1) {
  println("===== Transaction successful, buyer finalize the transaction =====")
  // add one week to the height
  addToHeight(720 * 7)

  // We will create an output box which withdraws the funds for the seller
  // We also need to subtract MinTxFee from value to account for tx fee we paid earlier
  val sellerWithdrawBox      = Box(value = depositAmount - MinTxFee,
                                   script = contract(seller.wallet.getAddress.pubKey))

  // Now we create the withdraw transaction
  // We set the input as the previous transaction
  val sellerWithdrawTransaction = Transaction(
        inputs       = List(escrowTransactionSigned.outputs(0)),
        outputs      = List(sellerWithdrawBox),
        fee          = MinTxFee,
        sendChangeTo = seller.wallet.getAddress
      )

  // The buyer sign the withdraw transaction to finalize the payment
  val sellerWithdrawTransactionSigned = buyer.wallet.sign(sellerWithdrawTransaction)

  // Submit the withdraw transaction to the mockchain
  blockchainSim.send(sellerWithdrawTransactionSigned)
}

// In this scenario, the buyer did not get the good after 4 weeks (over the 3 weeks)
// and ask the validator to proceed to a refund
if (chooser == 2) {
  println("===== Transaction refund, validator force the refund to the buyer after 4 weeks =====")
  addToHeight(720 * 7 * 4)
  // create a transaction for the buyer
  val buyerWithdrawBox = Box(value = depositAmount - MinTxFee,
                             script = contract(buyer.wallet.getAddress.pubKey))
  val buyerWithdrawTransaction = Transaction(
        inputs       = List(escrowTransactionSigned.outputs(0)),
        outputs      = List(buyerWithdrawBox),
        fee          = MinTxFee,
        sendChangeTo = buyer.wallet.getAddress
      )
  // The validator sign the withdraw transaction
  val buyerWithdrawTransactionSigned = validator.wallet.sign(buyerWithdrawTransaction)
  blockchainSim.send(buyerWithdrawTransactionSigned)
}

if (chooser == 3) {
  // In this scenario, the seller did not get the fund after 4 weeks (over the 3 weeks period)
  // and ask the validator to proceed to the payment as the good was delivered
  println("===== Transaction processes, validator force the payment to the seller after 4 weeks =====")
  addToHeight(720 * 7 * 4)
  // create a transaction for the seller
  val sellerWithdrawBox2 = Box(value = depositAmount - MinTxFee,
                               script = contract(seller.wallet.getAddress.pubKey))
  val sellerWithdrawTransaction2 = Transaction(
        inputs       = List(escrowTransactionSigned.outputs(0)),
        outputs      = List(sellerWithdrawBox2),
        fee          = MinTxFee,
        sendChangeTo = seller.wallet.getAddress
      )
  // The validator sign the withdraw transaction
  val sellerWithdrawTransactionSigned2 = validator.wallet.sign(sellerWithdrawTransaction2)
  blockchainSim.send(sellerWithdrawTransactionSigned2)
}

if (chooser == 4) {
  // In this scenario, the seller was not able to send the good after 2 week and refund the buyer
  println("===== Transaction refund, seller refund to the buyer after 2 weeks =====")
  addToHeight(720 * 7 * 4)
  // create a transaction for the buyer
  val buyerWithdrawBox2 = Box(value = depositAmount - MinTxFee,
                             script = contract(buyer.wallet.getAddress.pubKey))
  val buyerWithdrawTransaction2 = Transaction(
        inputs       = List(escrowTransactionSigned.outputs(0)),
        outputs      = List(buyerWithdrawBox2),
        fee          = MinTxFee,
        sendChangeTo = buyer.wallet.getAddress
      )
  // The seller sign the withdraw transaction
  val buyerWithdrawTransactionSigned2 = seller.wallet.sign(buyerWithdrawTransaction2)
  blockchainSim.send(buyerWithdrawTransactionSigned2)
}

// Print the users wallets, which shows the nanoErg have been sent (with same overall total as the initial amount minus the MinTxFee * 2)
buyer.printUnspentAssets()
seller.printUnspentAssets()
validator.printUnspentAssets()
println("---------------------")


// define useful function
def addToHeight( a:Int ) : Int = {
      val new_height = blockchainSim.getHeight+a
      blockchainSim.setHeight(new_height)
      println("New height: "+new_height)
      return new_height
   }


```
