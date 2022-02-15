Stealth Address
=================================

* Author: JD
* Created: Feb 13 2022
* License: CC0
* Difficulty: Expert
* Ergo Playground Link: [Stealth Address on Ergo by Example](https://scastie.scala-lang.org/bGs19JZnRKC5LQQxtjbdFA)

Description
----------
This contract shows how to create transactions that involve 
[stealth addresses](https://hackernoon.com/blockchain-privacy-enhancing-technology-series-stealth-address-i-c8a3eb4e4e43).

The contract builds on the solution suggested on the 
[Ergo forum by scalahub](https://www.ergoforum.org/t/stealth-address-contract/255).

This contract shows how to rely on ProveDHTuple as well as the need to sign transactions that have proveDHTuple with
the DHTuple secret.


Code
----------
#### [Stealth Address on Ergo by Example](https://scastie.scala-lang.org/bGs19JZnRKC5LQQxtjbdFA)
```scala

import java.math.BigInteger
import scorex.utils.Random
import sigmastate.basics.ProveDHTuple
import sigmastate.eval._
import sigmastate.interpreter.CryptoConstants.{EcPointType, dlogGroup}
import special.sigma.GroupElement

import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
import org.ergoplatform.ErgoBox

def makeStealthAddressBoxFor(
                              receiverAddress: Address,
                              receivingFunds: Long) : org.ergoplatform.ErgoBoxCandidate = {
  val DHTupleScript = s"""
    {
      val g_r1 = SELF.R4[GroupElement].get
      val g_r2 = SELF.R5[GroupElement].get
      val g_r1_s = SELF.R6[GroupElement].get
      val g_r2_s = SELF.R7[GroupElement].get

      proveDHTuple(g_r1,g_r2,g_r1_s,g_r2_s)
    }
    """.stripMargin
  val stealthAddressContract = ErgoScriptCompiler.compile(Map(), DHTupleScript)

  def makeARandomDHTupleProvableBy(pubKey: GroupElement) :
  (GroupElement, GroupElement, GroupElement, GroupElement) = {
    def makeRandomCustomGeneratorAndEmbedSecretFor(pubKey: GroupElement) :
    (GroupElement, GroupElement) = {
      val g : GroupElement = dlogGroup.generator
      // Large positive number of 32 bytes = Ergo 'private key'.
      val r = new BigInteger(1, Random.randomBytes(32))


      val rCustomGenerator = g.exp(r)
      val embeddedSecretOnRCustomGenerator = pubKey.exp(r) // works since g^ab = g^ba

      (rCustomGenerator, embeddedSecretOnRCustomGenerator)
    }

    val g_s = pubKey

    val (g_r1, g_r1_s) = makeRandomCustomGeneratorAndEmbedSecretFor(g_s)
    val (g_r2, g_r2_s) = makeRandomCustomGeneratorAndEmbedSecretFor(g_s)

    (g_r1, g_r2, g_r1_s, g_r2_s)
  }

  val receiverGroupElement : GroupElement = receiverAddress.proveDlog.h

  val (g_r1, g_r2, g_r1_s, g_r2_s) = makeARandomDHTupleProvableBy(receiverGroupElement)

  val stealthAddressRegisters:Map[ErgoBox.NonMandatoryRegisterId, Any] = Map(
    R4 -> g_r1,
    R5 -> g_r2,
    R6 -> g_r1_s,
    R7 -> g_r2_s
  )

  val stealthAddressBox = Box(value = receivingFunds,
    registers = stealthAddressRegisters,
    script = stealthAddressContract
  )
  stealthAddressBox
}

def unpackBoxForDHTuple(box: ErgoBox) : ProveDHTuple = {
  val gv = box.R4[GroupElement].get
  val hv = box.R5[GroupElement].get
  val uv = box.R6[GroupElement].get
  val vv = box.R7[GroupElement].get
  ProveDHTuple(gv,hv,uv,vv)
}


///////////////////////////////////////////////////////////////////////////////////
// Prepare A Test Scenario //
///////////////////////////////////////////////////////////////////////////////////
// Create a simulated blockchain (aka "Mockchain")
val blockchainSim = newBlockChainSimulationScenario("Stealth address scenario 😎")
// Create simulated sender and receiver
val alice = blockchainSim.newParty("Alice")
val bob = blockchainSim.newParty("Bob")
// Defining the amount of nanoergs in an erg, making working with amounts easier
val nanoergsInErg = 1000000000L
val aliceInitialFunds = 3 * nanoergsInErg // 3 Ergs
alice.generateUnspentBoxes(toSpend = aliceInitialFunds)
// Set up complete. //
alice.printUnspentAssets()
bob.printUnspentAssets()
println("Set up complete.")
println()
///////////////////////////////////////////////////////////////////////////////////

// Let's say Alice wants to move her funds to a stealth address
val aliceStealthAddressBox = makeStealthAddressBoxFor(
  receiverAddress = alice.wallet.getAddress,
  receivingFunds = aliceInitialFunds - MinTxFee)
val sendToAliceStealthAddressTransaction = Transaction(
  inputs = alice.selectUnspentBoxes(toSpend = aliceInitialFunds),
  outputs = List(aliceStealthAddressBox),
  fee = MinTxFee,
  sendChangeTo = alice.wallet.getAddress
)

val sendToAliceStealthAddressTransactionSigned = alice.wallet.sign(sendToAliceStealthAddressTransaction)
// Scenario: Moving assets to Alice's stealth address.
blockchainSim.send(sendToAliceStealthAddressTransactionSigned)
println("Notice: how it says Alice has no funds!")
alice.printUnspentAssets()
bob.printUnspentAssets()
println()

// Now Alice wants to move her funds to bob's stealth address
val aliceStealthAddressOutputBox = sendToAliceStealthAddressTransactionSigned.outputs(0)
val bobStealthAddressBox = makeStealthAddressBoxFor(
  receiverAddress = bob.wallet.getAddress,
  receivingFunds = aliceInitialFunds - 2 * MinTxFee)
val sendToBobStealthAddressTransaction = Transaction(
  inputs = List(aliceStealthAddressOutputBox),
  outputs = List(bobStealthAddressBox),
  fee = MinTxFee,
  sendChangeTo = alice.wallet.getAddress
)

val aliceStealthAddressOutputBoxProveDHTuple = unpackBoxForDHTuple(aliceStealthAddressOutputBox)

// Scenario: Moving assets to Bob's stealth address.
val sendToBobStealthAddressTransactionSigned = alice.wallet.withDHSecret(
  aliceStealthAddressOutputBoxProveDHTuple
).sign(sendToBobStealthAddressTransaction)
blockchainSim.send(sendToBobStealthAddressTransactionSigned)
println("Notice: how it says Bob & Alice have no funds!")
alice.printUnspentAssets()
bob.printUnspentAssets()
println()


// Just for experimentation sake let's send it to Bob's real address now.
val bobStealthAddressOutputBox = sendToBobStealthAddressTransactionSigned.outputs(0)
val bobAddressBox = Box(value = aliceInitialFunds - 3 * MinTxFee,
  script = contract(bob.wallet.getAddress.pubKey))
val sendToBobAddressTransaction = Transaction(
  inputs = List(bobStealthAddressOutputBox),
  outputs = List(bobAddressBox),
  fee = MinTxFee,
  sendChangeTo = bob.wallet.getAddress
)

val bobStealthAddressOutputBoxProveDHTuple = unpackBoxForDHTuple(bobStealthAddressOutputBox)
val sendToBobAddressTransactionSigned = bob.wallet.withDHSecret(
  bobStealthAddressOutputBoxProveDHTuple
).sign(sendToBobAddressTransaction)

// End Scenario: Moving assets to Bob's regular address.
blockchainSim.send(sendToBobAddressTransactionSigned)
println("Notice: how it says Bob has funds!")
alice.printUnspentAssets()
bob.printUnspentAssets()

println("Transaction scenario complete.")

```