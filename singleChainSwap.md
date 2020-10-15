Pin Lock Contract
=================================

* Author: Alexander Chepurnoy (aka kushti)
* Created: October 15 2020
* License: CC0
* Ergo Playground Link: [Single Chain Swap Contract](https://scastie.scala-lang.org/nITjDPcjTzm79SClVAWaNg)

In this example, we consider simplest single-chain swap contracts for buying and selling tokens for ergs.
The swap contracts will be developed into complex DEX scenarios in follow-up examples.

Consider, for example, Alice and Bob who want to exchange tokens: they agree that Alice (also, the seller) will give 
Bob (also, the buyer) 60 tokens of type TKN in exchange for 100 Ergo tokens. As protocol can be aborted by any party,
both also are willing to have possibility to refund value locked in swap order contracts.

In the interactive example above, we're testing swap between Alice and Bob, as well as a refund (for Bob's order only).

Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/nITjDPcjTzm79SClVAWaNg)


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
val blockchainSim = newBlockChainSimulationScenario("Single-Chain Swap Scenario")

val token = blockchainSim.newToken("TKN")
val tokenAmount = 60L


// Define a buyer Bob (with a wallet)
val buyer = blockchainSim.newParty("buyer Bob")

// Define a seller Alice (with a wallet tied)
val seller = blockchainSim.newParty("seller Alice")

///////////////////////////////////////////////////////////////////////////////////
// Single-chain Swap Contracts - Buy Contract                                    //
///////////////////////////////////////////////////////////////////////////////////

// buyerPK part is refund path
// Main path ensures that the box can be spent only in a transaction that produces an output with 60 tokens of type TKN and gives them to Bob 
// (Bob can reclaim the box after). Moreover, the last condition (OUTPUTS(0).R4[Col[Byte]].get == SELF.id) ensures that if Boe has multiple such boxes 
// outstanding at a given time, each will produce a separate output that identifies the corresponding input.  
// This condition prevents the following attack: if Bob has twosuch boxes outstanding but the last condition is not present, then they can be both used in a single
// transaction that contains just one output with 60 tokens of type “TKN” — the script of each input box will be individually satisfied, but Bob will get only 
// half of what owed to him.

val buyScript = s"""
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

val buyTokensContract = ErgoScriptCompiler.compile(Map("tokenId" -> token.tokenId, 
                                                       "tokenAmount" -> tokenAmount,
                                                       "buyerPk" -> buyer.wallet.getAddress.pubKey
                                                      ), buyScript)

// Define initial buyer Bob's balance
val nanoergsInErg = 1000000000L 
val buyerFunds = 200 * nanoergsInErg + (nanoergsInErg / 10) 

// Generate initial userFunds in the user's wallet
buyer.generateUnspentBoxes(toSpend = buyerFunds)
buyer.printUnspentAssets()
println("-----------")

val buyOrderSize = 100 * nanoergsInErg

///////////////////////////////////////////////////////////////////////////////////
// Create Buy Order //
///////////////////////////////////////////////////////////////////////////////////
// Create an output box with the user's funds locked under the contract
val buyOrderBox      = Box(value = buyOrderSize, 
                           script = buyTokensContract)

// Create the transaction which locks the buyer funds under the buy order contract
val buyOrderTransaction = Transaction(
      inputs       = buyer.selectUnspentBoxes(toSpend = buyerFunds),
      outputs      = List(buyOrderBox),
      fee          = MinTxFee,
      sendChangeTo = buyer.wallet.getAddress
    )

// Sign the buyOrderTransaction
val buyOrderTransactionSigned = buyer.wallet.sign(buyOrderTransaction)

// Submit the tx to the simulated blockchain
blockchainSim.send(buyOrderTransactionSigned)
buyer.printUnspentAssets()
println("-----------")


///////////////////////////////////////////////////////////////////////////////////
// Create Sell Order //
///////////////////////////////////////////////////////////////////////////////////

val sellerFunds = 2 * nanoergsInErg

// Generate initial userFunds in the user's wallet
seller.generateUnspentBoxes(toSpend = sellerFunds, tokensToSpend = List(token -> tokenAmount))
seller.printUnspentAssets()


// Sell order contract is very similar to buy order, just another output being used, and it should contain at least 100 ergs (100B nanoergs).   
val sellScript = s"""
  {
  val defined = OUTPUTS(0).R2[Coll[(Coll[Byte], Long)]].isDefined && OUTPUTS(0).R4[Coll[Byte]].isDefined
    sellerPk || sigmaProp (if (defined) {
              allOf(Coll(
                OUTPUTS(1).value >= 100000000000L,
                OUTPUTS(1).propositionBytes == sellerPk.propBytes,
                OUTPUTS(1).R4[Coll[Byte]].get == SELF.id
              ))
    } else { false } )         
  }
""".stripMargin

val sellTokensContract = ErgoScriptCompiler.compile(Map("sellerPk" -> seller.wallet.getAddress.pubKey), sellScript)


// Create a sell order box with the Alice's funds locked under the contract
val sellOrderBox      = Box(value = 1 * nanoergsInErg, 
                            token  = (token -> tokenAmount),
                            script = sellTokensContract)
///////////////////////////////////////////////////////////////////////////////////


// Create the sell order transaction which locks the users funds under the contract
val sellOrderTransaction = Transaction(
      inputs       = seller.selectUnspentBoxes(toSpend = sellerFunds),
      outputs      = List(sellOrderBox),
      fee          = MinTxFee,
      sendChangeTo = seller.wallet.getAddress
    )

// Print sellOrderTransaction
println(sellOrderTransaction)

// Sign the sellOrderTransaction
val sellOrderTransactionSigned = seller.wallet.sign(sellOrderTransaction)


// Submit the tx to the simulated blockchain
blockchainSim.send(sellOrderTransactionSigned)
seller.printUnspentAssets()
println("-----------")

// We flip a random coin and the whether doing refund (for the buyer's order only for simplicity's sake), or do the swap.
val refund = scala.util.Random.nextBoolean

if (refund) {
  println("=====Generating Refund for Buy Order")
  
  val buyBox = buyOrderTransactionSigned.outputs(0)
  
  val amt = buyBox.value
  
  val fee = 1000000L
  
  val refundOut = Box(value = amt - fee, 
                      script = contract(buyer.wallet.getAddress.pubKey))

  val refundTransaction = Transaction(
      inputs = List(buyBox),
      outputs = List(refundOut),
      sendChangeTo = buyer.wallet.getAddress,
      fee     = 1000000L
    )

  println(refundTransaction)

  val refundTransactionSigned = buyer.wallet.sign(refundTransaction)

  // Submit the tx to the simulated blockchain
  blockchainSim.send(refundTransactionSigned)
  buyer.printUnspentAssets()
  seller.printUnspentAssets()
  println("-----------")
  
} else {
  ///////////////////////////////////////////////////////////////////////////////////
  // Matching Orders //
  ///////////////////////////////////////////////////////////////////////////////////

  val tknOut = Box(value = nanoergsInErg / 2, 
                 token  = (token -> tokenAmount),
                 register = (R4 -> buyOrderTransactionSigned.outputs(0).id),
                 script = contract(buyer.wallet.getAddress.pubKey))

  val ergOut = Box(value = 100 * nanoergsInErg, 
                 register = (R4 -> sellOrderTransactionSigned.outputs(0).id),
                 script = contract(seller.wallet.getAddress.pubKey))

  val swapTransaction = Transaction(
      inputs = List(sellOrderTransactionSigned.outputs(0), buyOrderTransactionSigned.outputs(0)),
      outputs = List(tknOut, ergOut),
      sendChangeTo = buyer.wallet.getAddress,
      fee     = 1000000L
    )

  println(swapTransaction)

  val swapTransactionSigned = buyer.wallet.sign(swapTransaction)

  // Submit the tx to the simulated blockchain
  blockchainSim.send(swapTransactionSigned)
  buyer.printUnspentAssets()
  seller.printUnspentAssets()
  println("-----------")
}

```