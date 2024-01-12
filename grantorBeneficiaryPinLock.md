Grantor/Beneficiary Pin Lock Contract
=================================

* Author: PazuInTheSky
* Created: September 26 2021
* License: CC0
* Difficulty: Beginner
* Ergo Playground Link: [Grantor/Beneficiary Pin Lock Contract](https://scastie.scala-lang.org/wGP2oQ3ZQSSV5dqhOn7auA)

Description
----------

This contract is built on top of the [Pin Lock Contract](https://github.com/ergoplatform/ergoscript-by-example/blob/main/pinLockContract.md) to illustrate how you can start creating more complex flows:

* A _grantor_ deposits funds locked under a pin shared off-chain with a _beneficiary_ who is also able to withdraw the funds
* Only the _grantor_ or the _beneficiary_ can withdraw funds. The code includes an example you can uncomment to see how an attacker would fail to withdraw the funds without a compromised secret key of the _grantor_ or _beneficiary_, even with the pin number.
* the _grantor_ or the _beneficiary_ can withdraw funds without the approval of the other party.

Sigma-protocols are native to Ergo ([ErgoScript Whitepaper](https://ergoplatform.org/docs/ErgoScript.pdf)) and allow much more complex signature schemes such as multi-signature, that are impossible or cumbersome with most existing blockchain technology.
For instance, we might modify this contract to only let the _beneficiary_ remove funds with the approval of the _grantor_ only up to a certain amount and before a certain date/block height, _and_ the approval of a new _trustee_ (custodian) as a third party above this amount or past this date.

_Despite the additional proof `grantorPk || beneficiaryPk`, note that this contract is still intended to be used as an educational example and should not be used on-chain._

Code
----------

#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/vhkNicsSSgaNrFYtSQWQnw)

```scala
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
import sigmastate.eval.Extensions._
import scorex.crypto.hash.{Blake2b256}

///////////////////////////////////////////////////////////////////////////////////
// Prepare A Test Scenario                                                       //
///////////////////////////////////////////////////////////////////////////////////
// Create a simulated blockchain
val blockchainSim = newBlockChainSimulationScenario("Grantor/Beneficiary Pin Lock Scenario")

// Define a grantor (with a wallet)
// equivalent to "userParty" or "buyer" in simple PinLock Scenario that this scenario extends.
// https://github.com/ergoplatform/ergoscript-by-example/blob/main/pinLockContract.md
val grantorParty = blockchainSim.newParty("grantor")

// Beneficiary party is going to be whitelisted by grantor to withdraw funds with Pin number,
// as we’ll see ensure below (the same way as in simple PinLock scenario)
val beneficiaryParty = blockchainSim.newParty("beneficiary")

// Add an attacker for more fun
val unknownParty = blockchainSim.newParty("attacker")

// Define example grantor pin input (expected to be securely communicated to beneficiary off-chain)
val pinNumber = "1235813"
val hashedPin = Blake2b256(pinNumber.getBytes())

// Define initial nanoErgs in the grantor's wallet
// Note that `L` stands for `Long` integer in Scala
val nanoErgsInErg = 1000000000L // = 1 ERG
val grantorFunds = nanoErgsInErg + MinTxFee * 2 // 1 ERG + funds for fees of two transactions
// Generate initial grantorFunds in the grantor's wallet
grantorParty.generateUnspentBoxes(toSpend = grantorFunds)
grantorParty.printUnspentAssets()

// The beneficiary has no starting funds
beneficiaryParty.generateUnspentBoxes(toSpend = 0L)
beneficiaryParty.printUnspentAssets()

// nor has the attacker
unknownParty.generateUnspentBoxes(toSpend = 0L)
unknownParty.printUnspentAssets()

println("-----------")

///////////////////////////////////////////////////////////////////////////////////
// Create Grantor/Beneficiary Pin Lock Contract                                  //
///////////////////////////////////////////////////////////////////////////////////
// Create a Pin Lock script to let the specific beneficiary or the grantor herself
// submit a Pin number to withdraw funds.

// (grantorPk || beneficiaryPk) 
// This expression ensures that only the grantor or the beneficiary can withdraw funds,
// as long as their secret keys are not compromised,
// using native sigma-protocol in ErgoScript (https://ergoplatform.org/docs/ErgoScript.pdf).
// We use OR operator `||` so that only one private key is necessary to validate the contract,
// which entails that the beneficiary can withdraw funds without asking the grantor
// as long as the beneficiary knows the Pin number.
// Note that we use parentheses to enforce this one-of-two signature AND the following pin check.
// Without parentheses, the grantor would be able to withdraw funds without the pin.

// sigmaProp(SELF.R4[Coll[Byte]].get == blake2b256(OUTPUTS(0).R4[Coll[Byte]].get))
// Please refer to simple pin lock contract for details about this expression
// that checks the hash of the pin number.

val pinLockScript = s"""
  (grantorPk || beneficiaryPk) && sigmaProp(SELF.R4[Coll[Byte]].get == blake2b256(OUTPUTS(0).R4[Coll[Byte]].get))
""".stripMargin

// Compile a tailor-made contract with a `Map` of key to values required by pinLockScript
val pinLockContract = ErgoScriptCompiler.compile(Map("grantorPk" -> grantorParty.wallet.getAddress.pubKey,
                                                     "beneficiaryPk" -> beneficiaryParty.wallet.getAddress.pubKey),
                                                 pinLockScript)

///////////////////////////////////////////////////////////////////////////////////
// Deposit Funds Into Grantor/Beneficiary Pin Lock Contract                      //
///////////////////////////////////////////////////////////////////////////////////
println("GRANTOR DEPOSITS FUNDS")
// Create an output box with the grantor's funds (0.5 ERG + fees) locked under the contract
val fundsLocked = nanoErgsInErg / 2
val deposit     = fundsLocked + MinTxFee
val pinLockBox  = Box(value    = deposit,
                      script   = pinLockContract,
                      register = (R4 -> hashedPin))
// Create the deposit transaction which locks the grantor's funds under the contract
val depositTransaction = Transaction(
      inputs       = grantorParty.selectUnspentBoxes(toSpend = grantorFunds),
      outputs      = List(pinLockBox),
      fee          = MinTxFee,
      sendChangeTo = grantorParty.wallet.getAddress
    )

println(depositTransaction)

// Sign the depositTransaction
val depositTransactionSigned = grantorParty.wallet.sign(depositTransaction)

// Submit the tx to the simulated blockchain
blockchainSim.send(depositTransactionSigned)

grantorParty.printUnspentAssets()
println("Grantor has deposited %d nanoERGs and paid a %d nanoERG fee.".format(deposit, MinTxFee))
println("-----------")

///////////////////////////////////////////////////////////////////////////////////
// An unknown attacker attempts to withdraw funds locked Under Pin Lock Contract  //
///////////////////////////////////////////////////////////////////////////////////
/*
// NOTE: Uncomment this attack section to test proof in contract
println("ATTACK")

val unprovenWithdrawBox = Box(value    = fundsLocked,
                              // Simulate an attack trying to withdraw locked funds
                              // without the attacker knowing grantor or beneficiary secret key
                              script   = contract(unknownParty.wallet.getAddress.pubKey),
                              register = (R4 -> pinNumber.getBytes()))

val unprovenWithdrawTx = Transaction(inputs       = List(depositTransactionSigned.outputs(0)),
                                     outputs      = List(unprovenWithdrawBox),
                                     fee          = MinTxFee,
                                     sendChangeTo = unknownParty.wallet.getAddress)

println(unprovenWithdrawTx)

// Try to sign the unprovenWithdrawTx as the attacker
// A proof error should occur here
val unprovenWithdrawTxSigned = unknownParty.wallet.sign(unprovenWithdrawTx)

// Submit the unprovenWithdrawTx
blockchainSim.send(unprovenWithdrawTxSigned)

// No funds received due to error in contract
unknownParty.printUnspentAssets()

println("END OF ATTACK")
println("-----------")
*/

///////////////////////////////////////////////////////////////////////////////////
// Beneficiary withdraws half of funds locked under Grantor Pin Lock Contract    //
///////////////////////////////////////////////////////////////////////////////////
println("BENEFICIARY WITHDRAWS")
// Create an output box for the beneficiary to withdraw the funds

val partialWithdrawBox = Box(value    = fundsLocked / 2, // excluding fees
                             script   = contract(beneficiaryParty.wallet.getAddress.pubKey),
                             register = (R4 -> pinNumber.getBytes()))

// Create the partialWithdrawTransaction
val partialWithdrawTransaction = Transaction(
      inputs       = List(depositTransactionSigned.outputs(0)),
      outputs      = List(partialWithdrawBox),
      fee          = MinTxFee,
      // We could create a new output box with the same smart contract
      // to keep funds available to the beneficiary later on
      sendChangeTo = grantorParty.wallet.getAddress
    )

println(partialWithdrawTransaction)

// Sign the partialWithdrawTransaction
val partialWithdrawTransactionSigned = beneficiaryParty.wallet.sign(partialWithdrawTransaction)

// Submit the partialWithdrawTransaction
blockchainSim.send(partialWithdrawTransactionSigned)

println("-----------")
println("FINAL STATE")
// The grantor got back unused half of deposit
grantorParty.printUnspentAssets()
// The beneficiary now has 0.25 ERG
beneficiaryParty.printUnspentAssets()
println("-----------")


```
