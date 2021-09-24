Double Chain Swap
=================================

- Author: Keith Lim
- Created: September 12 2021
- License: CC0
- Difficulty: Intermediate
- Ergo Playground Link: [Double Chain Basic Swap Contract](https://scastie.scala-lang.org/b7rvMvR4QfecbJhIiu8Xsg)

Description
-----------

In this example, we explore the double-chain swap contracts which allow swapping of different tokens using ergoscript.

For our example at hand, consider a scenario where we have two hodlers, FstHodler and SndHodler, who wants to exchange their tokens for the other token.
They agree that FstHodler will swap with SndHodler 60 tokens of type FST in exchange for 40 tokens of type SND.

In the example below, we explore how to do a swap between FstHodler and SndHodler.

Code
----------

#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/b7rvMvR4QfecbJhIiu8Xsg)

```scala
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
import org.ergoplatform.Pay2SAddress
import sigmastate.eval.Extensions._
import scorex.crypto.hash.{Blake2b256}


///////////////////////////////////////////////////////////////////////////////////
// Double-chain Basic Swap Contracts                                            //
//////////////////////////////////////////////////////////////////////////////////
//////////////////////////////////////////////////////////////////////////////////
// Prepare A Test Scenario                                                      //
//////////////////////////////////////////////////////////////////////////////////
// Create a simulated blockchain (aka "Mockchain")
val blockchainSim = newBlockChainSimulationScenario("Double-Chain Swap Scenario")

// Create tokens.
val fstToken = blockchainSim.newToken("FST")
val fstTokenAmount = 60L

val sndToken = blockchainSim.newToken("SND")
val sndTokenAmount = 40L

// Define a swapper FST Hodler (with a wallet)
val fstHodler = blockchainSim.newParty("FST Hodler")

// Define a swapper SND Hodler (with a wallet)
val sndHodler = blockchainSim.newParty("SND Hodler")

///////////////////////////////////////////////////////////////////////////////////
// Swap Contract                                                                 //
///////////////////////////////////////////////////////////////////////////////////
// The main spending path ensures that the box can be spent only in a transaction that produces an
// output with N number of tokens of type X and gives them to the delegated swapper.
//
// val defined = OUTPUTS(0).R4[Coll[Byte]].isDefined
// This line ensures that R4 of outputs(0) is defined and is not null
//
// OUTPUTS(0).tokens(0)._1 == tokenId, OUTPUTS(0).tokens(0)._2 >= tokenAmount
// This line ensures the amount that is allowed to be swapped has to be of a specific token of tokenId
// and that the amount to be swapped has to be more than or equal to tokenAmount
//
// OUTPUTS(0).propositionBytes == swapperPk.propBytes
// This is to check what script is guarding the box. In this example, we check that the first box in outputs
// can be spent by whoever knows secret key for buyerPk. Another often used technique will be to check if a
// box guarded by the same script that is currently executing ‘OUTPUTS(0).propositionBytes == SELF. propositonBytes‘ - GreenHat
//
// The last condition (OUTPUTS(0).R4[Col[Byte]].get == SELF.id)
// ensures that if Bob has multiple of such boxes outstanding at a given time, each will produce
// a separate output that identifies the corresponding input.

val swapScript = s"""
  {
    val defined = OUTPUTS(0).R4[Coll[Byte]].isDefined
    swapperPk || sigmaProp (if (defined) {
      allOf(Coll(
          OUTPUTS(0).tokens(0)._1 == tokenId, OUTPUTS(0).tokens(0)._2 >= tokenAmount,
          OUTPUTS(0).propositionBytes == swapperPk.propBytes,
          OUTPUTS(0).R4[Coll[Byte]].get == SELF.id)
         )
    } else { false } )
  }
""".stripMargin

// Compile the contract with an included `Map` which specifies what the values of given parameters are going to be hard-coded into the contract
val sndTokensContract = ErgoScriptCompiler.compile(Map("tokenId" -> sndToken.tokenId,
                                                       "tokenAmount" -> sndTokenAmount,
                                                       "swapperPk" -> fstHodler.wallet.getAddress.pubKey
                                                      ), swapScript)
val fstTokensContract = ErgoScriptCompiler.compile(Map("tokenId" -> fstToken.tokenId,
                                                       "tokenAmount" -> fstTokenAmount,
                                                       "swapperPk" -> sndHodler.wallet.getAddress.pubKey
                                                      ), swapScript)

///////////////////////////////////////////////////////////////////////////////////
// Create Swap Order                                                             //
///////////////////////////////////////////////////////////////////////////////////

// Define initial hodler balance
val nanoergsInErg = MinTxFee
val fstHodlerFunds = 2 * nanoergsInErg
val sndHodlerFunds = 2 * nanoergsInErg

fstHodler.generateUnspentBoxes(
      toSpend       = fstHodlerFunds,
      tokensToSpend = List(fstToken -> fstTokenAmount))
fstHodler.printUnspentAssets()

// Create a fst order box with fstHodler's funds locked under the contract
// Note: the contract used in this has to be sndTokensContract because
// 1. the buyerPk for sndTokensContract is fstHodler's wallet address
// 2. the token to be sent to fstHodler's is SND (which is what fstHodler is requesting for the swap)
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

// Print Transaction
println(fstOrderTransaction)

// Sign the Transaction
val fstOrderTransactionSigned = fstHodler.wallet.sign(fstOrderTransaction)


// Submit the tx to the simulated blockchain
blockchainSim.send(fstOrderTransactionSigned)
fstHodler.printUnspentAssets()
println("-----------")

// Do the same for sndHodler
sndHodler.generateUnspentBoxes(
      toSpend       = sndHodlerFunds,
      tokensToSpend = List(sndToken -> sndTokenAmount))
sndHodler.printUnspentAssets()

// Create a snd order box with sndHodler's funds locked under the contract
// Note: the contract used in this has to be fstTokensContract because
// 1. the swapperPk for sndTokensContract is sndHodler's wallet address
// 2. the token to be sent to sndHodler's is FST (which is what sndHodler is requesting for the swap)
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

// Print the transaction
println(sndOrderTransaction)

// Sign the Transaction
val sndOrderTransactionSigned = sndHodler.wallet.sign(sndOrderTransaction)


// Submit the tx to the simulated blockchain
blockchainSim.send(sndOrderTransactionSigned)
sndHodler.printUnspentAssets()
println("-----------")


///////////////////////////////////////////////////////////////////////////////////
// Matching Orders //
///////////////////////////////////////////////////////////////////////////////////
// In this part, we will match the order and execute the swap.
println("===== Generating Order Matching Tx =====")


// Create a box for each holder to initiate the swap.
//
// token = (xToken -> xTokenAmount)
// is the token and the token amount that will be added to the order box.
// Reminder that ergo allows us to trade execute orders with tokens other than ergo.
//
// register = (R4 -> fstOrderTransactionSigned.outputs(0).id)
// This line inserts the previously signed transaction id into R4 of the box.
//
// script = contract(fstHodler.wallet.getAddress.pubKey)
// This line allows us to get the contract that is link to the hodler's public key.
val fstOut = Box(value  = nanoergsInErg / 2,
                token = (sndToken -> sndTokenAmount),
                register = (R4 -> fstOrderTransactionSigned.outputs(0).id),
                script = contract(fstHodler.wallet.getAddress.pubKey))

val sndOut = Box(value = nanoergsInErg / 2,
                token = (fstToken -> fstTokenAmount),
                register = (R4 -> sndOrderTransactionSigned.outputs(0).id),
                script = contract(sndHodler.wallet.getAddress.pubKey))


// Create the swap transaction where we will take in both previously signed transactions as inputs
// and the newly created order box as outputs.
// Ergo cannot be burned. For this example, we will just send it to the party that is signing
// the transaction (fstHodler).
val swapTransaction = Transaction(
    inputs = List(sndOrderTransactionSigned.outputs(0), fstOrderTransactionSigned.outputs(0)),
    outputs = List(fstOut, sndOut),
    sendChangeTo = fstHodler.wallet.getAddress,
    fee = MinTxFee
  )

// Print the transaction
println(swapTransaction)

// Sign the transaction
val swapTransactionSigned = fstHodler.wallet.sign(swapTransaction)


// Execute the transaction
blockchainSim.send(swapTransactionSigned)
fstHodler.printUnspentAssets()
sndHodler.printUnspentAssets()
println("------------")
```
