2.1 Sigma Statements
=================================

- Author: k.金お捕
- Created: October 31 2021
- License: CC0
- Difficulty: Beginner
- Ergo Playground Link: [Sigma Statements](https://scastie.scala-lang.org/ps9nmzynTKei4qZsUm7q3A)

Description
-----------

This is the examples that are provided in the Ergoscript document. In this example, we will cover sigma statements which includes usage of some scala code within the contracts.

This is best used to follow along with the Ergoscript document at [ErgoScript Whitepaper](https://ergoplatform.org/docs/ErgoScript.pdf). 

This example consists of 5 working examples, and 2 that has not been fully implemented yet. This is due to the requirement of multisig in the failing examples. You can run the examples by changing it to the specified enum in RunPkExample.

Code
----------

#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/ps9nmzynTKei4qZsUm7q3A)

```scala
import org.ergoplatform.ErgoLikeTransaction
import org.ergoplatform.playground.Party
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._

///////////////////////////////////////////
// 2.1 Sigma Contracts Scripts           //
///////////////////////////////////////////

val nanoErgsInErg = 1000000000L
val pkFunds = 100 * nanoErgsInErg

// Helper
def generateAndPrintUnspent(party: Party, funds: Long): Unit = {
  party.generateUnspentBoxes(toSpend = funds)
  party.printUnspentAssets()
}

object PKExamples extends Enumeration {
  type PKExamples = Value
  val PK,
    PKAPKB,
    PKAPKBPKC,
    AnyOfCollPKAPKBPKC,
    AtLeastThree,
    PKAAndPKB,
    PKAAndPKBOrPKC = Value
}

import PKExamples._

def RunPKExample(example: PKExamples) = {
  example match {
    case PKExamples.PK =>
      println("Running pk Example")
      pkExample()
    case PKExamples.PKAPKB =>
      println("Running pkA OR pkB Example")
      pkApkBExample()
    case PKExamples.PKAPKBPKC =>
      println("Running pkA OR pkB OR pkC Example")
      pkApkBpkCExample()
    case PKExamples.PKAAndPKB =>
      println("Running pkA AND pkB Example")
      pkAandpkBExample()
    case PKExamples.AnyOfCollPKAPKBPKC =>
      println("Running anyOf Coll pkA pkB pkC Example")
      anyOfCollpkApkBpkCExample()
    case PKExamples.PKAAndPKBOrPKC =>
      println("Running pkA AND pkB OR pkC Example")
      pkAandpkBorpkCExample()
    case PKExamples.AtLeastThree =>
      println("Running atLeast Three Example")
      atLeastThreeExample()
  }
}

RunPKExample(example = PKExamples.AnyOfCollPKAPKBPKC)



///////////////////////////////////////////
// 2.1 pk                                //
///////////////////////////////////////////
def pkExample() = {
  val blockchainSimulation = newBlockChainSimulationScenario("Sigma Statements: pk Example")
  // Define parties
  val pk = blockchainSimulation.newParty("person-0")
  val pkB = blockchainSimulation.newParty("person-B")
  generateAndPrintUnspent(party = pk, funds = pkFunds)
  println("---------------------------")

  val pkScript = s"""sigmaProp(pk)"""

  // Change the user wallet's pubkey here to see the changes
  val pkContract = ErgoScriptCompiler.compile(
    Map("pk" -> pk.wallet.getAddress.pubKey),
    pkScript)

  val pkBox = Box(
    value = pkFunds - MinTxFee,
    script = pkContract)

  val pkTx = Transaction(
    inputs = pk.selectUnspentBoxes(toSpend = pkFunds),
    outputs = List(pkBox),
    fee = MinTxFee,
    sendChangeTo = pk.wallet.getAddress
  )

  println(pkTx)
  println("------------------")

  val pkTxSigned = pk.wallet.sign(pkTx)
  blockchainSimulation.send(pkTxSigned)
  pk.printUnspentAssets()
  println("---------------")

  val didPkBSpent = false
  // If pkB spend, then it will fail
  val party = if (didPkBSpent) pkB else pk

  partyToSpend(party, pkTxSigned, blockchainSimulation)
}

/**
 * Selected party to spend the box
 * @param party
 * @param txSigned
 * @param blockchainSimulation
 */
def partyToSpend(party: Party, txSigned: ErgoLikeTransaction, blockchainSimulation: BlockchainSimulation) = {
  // party tries to spend the box
  val pkOutputBox = Box(
    value = pkFunds - 2 * MinTxFee,
    script = contract(party.wallet.getAddress.pubKey))

  val pkWithdrawTx = Transaction(
    inputs = List(txSigned.outputs(0)),
    outputs = List(pkOutputBox),
    fee = MinTxFee,
    sendChangeTo = party.wallet.getAddress
  )

  // If the script is rejected, an error will be thrown when signing.
  val pkWithdrawTxSigned = party.wallet.sign(pkWithdrawTx)

  blockchainSimulation.send(pkWithdrawTxSigned)

  party.printUnspentAssets()
}


///////////////////////////////////////////
//  pkA OR pkB                              //
///////////////////////////////////////////
def pkApkBExample() = {
  val blockchainSimulation = newBlockChainSimulationScenario("Sigma Statements: pkA || pkB Example")
  // Define parties
  val pkA = blockchainSimulation.newParty("person-A")
  val pkB = blockchainSimulation.newParty("person-B")
  generateAndPrintUnspent(party = pkA, funds = pkFunds)
  println("---------------------------")

  val pkApkBScript = s"""sigmaProp(pkA || pkB)"""

  // Change the user wallet's pubkey here to see the changes
  val pkContract = ErgoScriptCompiler.compile(
    Map("pkA" -> pkA.wallet.getAddress.pubKey,
      "pkB" -> pkB.wallet.getAddress.pubKey),
    pkApkBScript)

  val pkBox = Box(
    value = pkFunds - MinTxFee,
    script = pkContract)

  val pkTx = Transaction(
    inputs = pkA.selectUnspentBoxes(toSpend = pkFunds),
    outputs = List(pkBox),
    fee = MinTxFee,
    sendChangeTo = pkA.wallet.getAddress
  )

  println(pkTx)
  println("------------------")

  val pkTxSigned = pkA.wallet.sign(pkTx)
  blockchainSimulation.send(pkTxSigned)
  pkA.printUnspentAssets()
  println("---------------")

  val didPkBSpent = true
  val party = if (didPkBSpent) pkB else pkA

  partyToSpend(party, pkTxSigned, blockchainSimulation)
}


///////////////////////////////////////////
//  pkA OR pkB OR pkC                    //
///////////////////////////////////////////
def pkApkBpkCExample() = {
  val blockchainSimulation = newBlockChainSimulationScenario("Sigma Statements: pkA || pkB || pkC Example")
  // Define parties
  val pkA = blockchainSimulation.newParty("person-A")
  val pkB = blockchainSimulation.newParty("person-B")
  val pkC = blockchainSimulation.newParty("person-C")
  generateAndPrintUnspent(party = pkA, funds = pkFunds)
  println("---------------------------")

  val pkApkBpkCScript = s"""sigmaProp(pkA || pkB || pkC)"""

  // Change the user wallet's pubkey here to see the changes
  val pkContract = ErgoScriptCompiler.compile(
    Map("pkA" -> pkA.wallet.getAddress.pubKey,
      "pkB" -> pkB.wallet.getAddress.pubKey,
      "pkC" -> pkC.wallet.getAddress.pubKey),
  pkApkBpkCScript)

  val pkBox = Box(
    value = pkFunds - MinTxFee,
    script = pkContract)

  val pkTx = Transaction(
    inputs = pkA.selectUnspentBoxes(toSpend = pkFunds),
    outputs = List(pkBox),
    fee = MinTxFee,
    sendChangeTo = pkA.wallet.getAddress
  )

  println(pkTx)
  println("------------------")

  val pkTxSigned = pkA.wallet.sign(pkTx)
  blockchainSimulation.send(pkTxSigned)
  pkA.printUnspentAssets()
  println("---------------")

  // Change parties here to see changes
  partyToSpend(pkC, pkTxSigned, blockchainSimulation)
}


///////////////////////////////////////////
//  anyOf Coll pkA pkB pkC               //
///////////////////////////////////////////
def anyOfCollpkApkBpkCExample() = {
  val blockchainSimulation = newBlockChainSimulationScenario("Sigma Statements: anyOf Coll pkA pkB pkC Example")
  // Define parties
  val pkA = blockchainSimulation.newParty("person-A")
  val pkB = blockchainSimulation.newParty("person-B")
  val pkC = blockchainSimulation.newParty("person-C")
  generateAndPrintUnspent(party = pkA, funds = pkFunds)
  println("---------------------------")

  val anyOfCollpkApkBpkCScript =
    s"""
       |{
       |  val anyOfpk = anyOf(Coll(pkA, pkB, pkC))
       |  sigmaProp(anyOfpk)
       |}
       |""".stripMargin

  // Change the user wallet's pubkey here to see the changes
  val pkContract = ErgoScriptCompiler.compile(
    Map("pkA" -> pkA.wallet.getAddress.pubKey,
      "pkB" -> pkB.wallet.getAddress.pubKey,
      "pkC" -> pkC.wallet.getAddress.pubKey),
    anyOfCollpkApkBpkCScript)

  val pkBox = Box(
    value = pkFunds - MinTxFee,
    script = pkContract)

  val pkTx = Transaction(
    inputs = pkA.selectUnspentBoxes(toSpend = pkFunds),
    outputs = List(pkBox),
    fee = MinTxFee,
    sendChangeTo = pkA.wallet.getAddress
  )

  println(pkTx)
  println("------------------")

  val pkTxSigned = pkA.wallet.sign(pkTx)
  blockchainSimulation.send(pkTxSigned)
  pkA.printUnspentAssets()
  println("---------------")

  // Change parties here to see changes
  partyToSpend(pkC, pkTxSigned, blockchainSimulation)
}


///////////////////////////////////////////
//  pkA AND pkB                          //
///////////////////////////////////////////
def pkAandpkBExample() = {
  val blockchainSimulation = newBlockChainSimulationScenario("Sigma Statements: pkA and pkB Example")
  // Define parties
  val pkA = blockchainSimulation.newParty("person-A")
  val pkB = blockchainSimulation.newParty("person-B")
  generateAndPrintUnspent(party = pkA, funds = pkFunds)
  println("---------------------------")

  val pkAandpkBScript =
    s"""{
       |  sigmaProp(pkA && pkB)
       |}""".stripMargin

  // Change the user wallet's pubkey here to see the changes
  val pkContract = ErgoScriptCompiler.compile(
    Map("pkA" -> pkA.wallet.getAddress.pubKey,
      "pkB" -> pkB.wallet.getAddress.pubKey),
    pkAandpkBScript)

  val pkBox = Box(
    value = pkFunds - MinTxFee,
    script = pkContract)

  val pkTx = Transaction(
    inputs = pkA.selectUnspentBoxes(toSpend = pkFunds),
    outputs = List(pkBox),
    fee = MinTxFee,
    sendChangeTo = pkA.wallet.getAddress
  )

  println(pkTx)
  println("------------------")

  val pkTxSigned = pkA.wallet.sign(pkTx)
  blockchainSimulation.send(pkTxSigned)
  pkA.printUnspentAssets()
  println("---------------")

  // Change parties here to see changes
  val twoPartiesSign = false

  if (twoPartiesSign) {
    val parties = Array(pkA, pkB)
    partiesSpend(parties, pkTxSigned, blockchainSimulation)
  }
  else
    partyToSpend(pkA, pkTxSigned, blockchainSimulation)
}

/**
 * Goes through all parties and sign the transaction then spend it
 * @todo ki, implement when Party allows multisig
 * https://github.com/ScorexFoundation/sigmastate-interpreter/blob/311407568589767b2fa0150fda836a6bc768eebc/sigmastate/src/test/scala/sigmastate/utxo/DistributedSigSpecification.scala#L8-L8
 * @param parties
 * @param txSigned
 * @param blockchainSimulation
 * @return
 */
def partiesSpend(parties: Array[Party], txSigned: ErgoLikeTransaction, blockchainSimulation: BlockchainSimulation) = {
  for (party <- parties) party.printUnspentAssets()
  throw new NotImplementedError()
}


///////////////////////////////////////////
//  pkA AND pkB OR pkC                   //
///////////////////////////////////////////
def pkAandpkBorpkCExample() = {
  val blockchainSimulation = newBlockChainSimulationScenario("Sigma Statements: pk && pkB || pkC Example")
  // Define parties
  val pkA = blockchainSimulation.newParty("person-A")
  val pkB = blockchainSimulation.newParty("person-B")
  val pkC = blockchainSimulation.newParty("person-C")
  generateAndPrintUnspent(party = pkA, funds = pkFunds)
  println("---------------------------")

  val pkAandpkBorpkCScript = s"""sigmaProp((pkA && pkB) || pkC)"""

  // Change the user wallet's pubkey here to see the changes
  val pkContract = ErgoScriptCompiler.compile(
    Map("pkA" -> pkA.wallet.getAddress.pubKey,
      "pkB" -> pkB.wallet.getAddress.pubKey,
      "pkC" -> pkC.wallet.getAddress.pubKey),
  pkAandpkBorpkCScript)

  val pkBox = Box(
    value = pkFunds - MinTxFee,
    script = pkContract)

  val pkTx = Transaction(
    inputs = pkA.selectUnspentBoxes(toSpend = pkFunds),
    outputs = List(pkBox),
    fee = MinTxFee,
    sendChangeTo = pkA.wallet.getAddress
  )

  println(pkTx)
  println("------------------")

  val pkTxSigned = pkA.wallet.sign(pkTx)
  blockchainSimulation.send(pkTxSigned)
  pkA.printUnspentAssets()
  println("---------------")

  // Change parties here to see changes
  val twoPartiesSign = false

  if (twoPartiesSign) {
    val parties = Array(pkA, pkB)
    partiesSpend(parties, pkTxSigned, blockchainSimulation)
  }
  else
    partyToSpend(pkC, pkTxSigned, blockchainSimulation)
}


///////////////////////////////////////////
//  atLeast Three                        //
///////////////////////////////////////////
def atLeastThreeExample() = {
  val blockchainSimulation = newBlockChainSimulationScenario("Sigma Statements: atLeast Three Example")
  // Define parties
  val pkA = blockchainSimulation.newParty("person-A")
  val pkB = blockchainSimulation.newParty("person-B")
  val pkC = blockchainSimulation.newParty("person-C")
  val pkD = blockchainSimulation.newParty("person-d")
  generateAndPrintUnspent(party = pkA, funds = pkFunds)
  println("---------------------------")

  val atLeastThreeScript =
    s"""
       |{
       |  val pks = Coll(pkA, pkB, pkC && pkD)
       |  val atLeastThree = (2, pks)
       |  sigmaProp(atLeastThree)
       |}
       |""".stripMargin

  // Change the user wallet's pubkey here to see the changes
  val pkContract = ErgoScriptCompiler.compile(
    Map("pkA" -> pkA.wallet.getAddress.pubKey,
      "pkB" -> pkB.wallet.getAddress.pubKey,
      "pkC" -> pkC.wallet.getAddress.pubKey,
      "pkD" -> pkD.wallet.getAddress.pubKey),
  atLeastThreeScript)

  val pkBox = Box(
    value = pkFunds - MinTxFee,
    script = pkContract)

  val pkTx = Transaction(
    inputs = pkA.selectUnspentBoxes(toSpend = pkFunds),
    outputs = List(pkBox),
    fee = MinTxFee,
    sendChangeTo = pkA.wallet.getAddress
  )

  println(pkTx)
  println("------------------")

  val pkTxSigned = pkA.wallet.sign(pkTx)
  blockchainSimulation.send(pkTxSigned)
  pkA.printUnspentAssets()
  print("---------------")

  // Change parties here to see changes
  val partiesSign = false

  if (partiesSign) {
    val parties = Array(pkA, pkB, pkC, pkD)
    partiesSpend(parties, pkTxSigned, blockchainSimulation)
  }
  else
    partyToSpend(pkC, pkTxSigned, blockchainSimulation)
}
```
