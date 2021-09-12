Single Chain Swap
=================================

* Author: Keith Lim (aka kushti)
* Created: September 12 2021
* License: CC0
* Difficulty: Intermediate
* Ergo Playground Link: [Double Chain Swap Contract](https://scastie.scala-lang.org/EPnXlipFRgaGRSL8mQf0mw)

Description
----------
In this example, we explore the double-chain swap contracts which allow swapping of different tokens using ergoscript.

For our example at hand, consider a scenario where FstHodler and SndHodler want to exchange their tokens.
They agree that FstHodler (the seller) will give Bob (the buyer) 60 tokens of type FST in exchange for 60 tokens of type SND.

In the interactive example below, we explore how to do a test swap between FstHodler and SndHodler.

Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/EPnXlipFRgaGRSL8mQf0mw)


```scala
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
import org.ergoplatform.Pay2SAddress
import sigmastate.eval.Extensions._ 
import scorex.crypto.hash.{Blake2b256}

///////////////////////////////////////////////////////////////////////////////////
// Prepare A Test Scenario //
///////////////////////////////////////////////////////////////////////////////////
// Create a simulated blockchain (aka "Mockchain")
val blockchainSim = newBlockChainSimulationScenario("Double-Chain Swap Scenario")

// Create a new token called "FST"
val fstToken = blockchainSim.newToken("FST")
val fstTokenAmount = 60L

// Create a new token called "SND"
val sndToken = blockchainSim.newToken("SND")
val sndTokenAmount = 60L

// Define a swapper Bob (with a wallet)
val fstHodler = blockchainSim.newParty("FST Hodler")

// Define a swapper Alice (with a wallet)
val sndHodler = blockchainSim.newParty("SND Hodler")

///////////////////////////////////////////////////////////////////////////////////
// Double-chain Swap Contracts - Swap Contract                                    //
///////////////////////////////////////////////////////////////////////////////////

// The main spending path ensures that the box can be spent only in a transaction that produces an output with 60 tokens of type TKN and gives them to Bob 
// (Bob can reclaim the box after). Moreover, the last condition (OUTPUTS(0).R4[Col[Byte]].get == SELF.id) ensures that if Bob has multiple of such boxes 
// outstanding at a given time, each will produce a separate output that identifies the corresponding input.  
// This condition prevents the following attack: if Bob has two of such boxes outstanding but the last condition is not present, then they can be both used in a single
// transaction that contains just one output with 60 tokens of type “TKN” — the script of each input box will be individually satisfied, but Bob will get only 
// half of what owed to him.
// The buyerPK check is the refund spending path.

val swapScript = s"""
  {
    val defined = OUTPUTS(0).R2[Coll[(Coll[Byte], Long)]].isDefined && OUTPUTS(0).R4[Coll[Byte]].isDefined
    buyerPk || sigmaProp (if (defined) {
      allOf(Coll(
          OUTPUTS(0).tokens(0)._1 == tokenId, OUTPUTS(0).tokens(0)._2 >= tokenAmount,
          OUTPUTS(0).propositionBytes == buyerPk.propBytes,
          OUTPUTS(0).R4[Coll[Byte]].get == SELF.id)
         )
    } else { false } )
  }
""".stripMargin

// Compile the contract with an included `Map` which specifies what the values of given parameters are going to be hard-coded into the contract
val sndTokensContract = ErgoScriptCompiler.compile(Map("tokenId" -> sndToken.tokenId, 
                                                       "tokenAmount" -> sndTokenAmount,
                                                       "buyerPk" -> fstHodler.wallet.getAddress.pubKey
                                                      ), swapScript)
val fstTokensContract = ErgoScriptCompiler.compile(Map("tokenId" -> fstToken.tokenId, 
                                                       "tokenAmount" -> fstTokenAmount,
                                                       "buyerPk" -> sndHodler.wallet.getAddress.pubKey
                                                      ), swapScript)

///////////////////////////////////////////////////////////////////////////////////
// Create Buy Order //
///////////////////////////////////////////////////////////////////////////////////

// Define initial hodler balance
val nanoergsInErg = 1000000000L 
val fstHodlerFunds = 2 * nanoergsInErg
val sndHodlerFunds = 2 * nanoergsInErg

fstHodler.generateUnspentBoxes(
      toSpend       = fstHodlerFunds, 
      tokensToSpend = List(fstToken -> fstTokenAmount))
fstHodler.printUnspentAssets()

// Create a fst order box with fstHodler's funds locked under the contract
val fstOrderBox   = Box(value = 1 * nanoergsInErg,
                        token = (fstToken -> fstTokenAmount),
                        script = sndTokensContract)

// Create the fst order transaction which locks the hodlers funds under the contract
val fstOrderTransaction = Transaction(
      inputs        = fstHodler.selectUnspentBoxes(toSpend = fstHodlerFunds),
      outputs       = List(fstOrderBox),
      fee           = MinTxFee,
      sendChangeTo  = fstHodler.wallet.getAddress
  )

// Print fstOrderTransaction
println(fstOrderTransaction)

// Sign the fstOrderTransaction
val fstOrderTransactionSigned = fstHodler.wallet.sign(fstOrderTransaction)


// Submit the tx to the simulated blockchain
blockchainSim.send(fstOrderTransactionSigned)
fstHodler.printUnspentAssets()
println("-----------")


sndHodler.generateUnspentBoxes(
      toSpend       = sndHodlerFunds, 
      tokensToSpend = List(sndToken -> sndTokenAmount))
sndHodler.printUnspentAssets()

// Create a snd order box with sndHodler's funds locked under the contract
val sndOrderBox   = Box(value = 1 * nanoergsInErg,
                        token = (sndToken -> sndTokenAmount),
                        script = fstTokensContract)

// Create the fst order transaction which locks the hodlers funds under the contract
val sndOrderTransaction = Transaction(
      inputs        = sndHodler.selectUnspentBoxes(toSpend = sndHodlerFunds),
      outputs       = List(fstOrderBox),
      fee           = MinTxFee,
      sendChangeTo  = sndHodler.wallet.getAddress
  )

// Print fstOrderTransaction
println(sndOrderTransaction)

// Sign the fstOrderTransaction
val sndOrderTransactionSigned = sndHodler.wallet.sign(sndOrderTransaction)


// Submit the tx to the simulated blockchain
blockchainSim.send(sndOrderTransactionSigned)
sndHodler.printUnspentAssets()
println("-----------")


///////////////////////////////////////////////////////////////////////////////////
// Matching Orders //
///////////////////////////////////////////////////////////////////////////////////
println("===== Generating Order Matching Tx =====")


// Create the box which goes back to fstHodler which holds the SND Tokens
val fstOut = Box(value  = nanoergsInErg / 2,
                token = (sndToken -> sndTokenAmount),
                register = (R4 -> fstOrderTransactionSigned.outputs(0).id),
                script = contract(fstHodler.wallet.getAddress.pubKey))

// Create the box which goes back to sndHodler which holds the FST Tokens
val sndOut = Box(value = nanoergsInErg / 2,
                token = (fstToken -> fstTokenAmount),
                register = (R4 -> sndOrderTransactionSigned.outputs(0).id),
                script = contract(sndHodler.wallet.getAddress.pubKey))

// Create the swap transaction
val swapTransaction = Transaction(
    inputs = List(sndOrderTransactionSigned.outputs(0), fstOrderTransactionSigned.outputs(0)),
    outputs = List(fstOut, sndOut),
    sendChangeTo = fstHodler.wallet.getAddress,
    fee = 1000000L
  )

println(swapTransaction)

val swapTransactionSigned = fstHodler.wallet.sign(swapTransaction)

// Submit the tx to the simulated blockchain
blockchainSim.send(swapTransactionSigned)
fstHodler.printUnspentAssets()
sndHodler.printUnspentAssets()
println("------------")
```
