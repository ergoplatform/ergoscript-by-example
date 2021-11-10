Loser pays
=================================

* Author: ladopixel
* Created: November 9 2021
* License: CC0
* Difficulty: (Beginner)
* Ergo Playground Link: [Loser pays](https://scastie.scala-lang.org/SqWeyRSLSNS46AxI4p1t0g)

Description
----------
Very simple game example.

In the scene there are 2 players and a waiter.
Each player receives the same amount of NanoErgs, while the waiter has 0.
A random number between 0 and 100 is generated, with which each player has a 50% chance of winning or losing.
The losing player sends his NanoErgs to the waiter as payment.

Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/SqWeyRSLSNS46AxI4p1t0g)
```scala

// I import the necessary for everything to work properly
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._

// I create some immutable variables and another mutable one
// Define scenario, players and quantities
val randomPay = scala.util.Random
val blockchainGame = newBlockChainSimulationScenario("game blockchain")
val player1 = blockchainGame.newParty("player1")
val player2 = blockchainGame.newParty("player2")
val waiter = blockchainGame.newParty("waiter")
val funds = 900000000

var winner = 0

// Sending to the players' wallet 900000000 nanoergs
player1.generateUnspentBoxes(toSpend = funds)
player2.generateUnspentBoxes(toSpend = funds)

// I print on the screen the current situation:
// Player 1: 900000000
// Player 2: 900000000
// Waiter: 0
println("------------------------")
println("The one who loses pays")
println("------------------------")
println("Starting balance")
player1.printUnspentAssets()
player2.printUnspentAssets()
waiter.printUnspentAssets()
println("---------------------")

// I urge the contract to be fulfilled
val trueScript = "sigmaProp(1 == 1)"
val trueContract = ErgoScriptCompiler.compile(Map(), trueScript)

val maximoParaEnviar = (funds - MinTxFee)

// I generate a random number less than 100, if randomPay > 50 the winner is player 2, if randomPay < 50 the winner is player 1
if(randomPay.nextInt(100) > 50) {
    winner = 2

    // We create the first box, blocking the funds to the contract and assigning values.
    val trueBox = Box(value = maximoParaEnviar, script = trueContract)

    // R0: NanoErgs - 1 Erg = 1000000000 NanoErgs
    // R1: The smart contract script
    // R2: Aditionals actives (generated automatically)
    // R3: txId (generated automatically)
    
    // Aditionals registers R4, R5, R6, R7, R8, R9 in a NFT Picture or Audio
    // R4: Name
    // R5: Description
    // R6: Decimals
    // R7: Picture 0e020101 - Audio 0e020102
    // R8: SHA256 
    // R9: Optional link to the artwork


    // We create the transaction taking into account that we will have inputs, outputs, fee and to return the change.
    val depositTransaction = Transaction(
        inputs       = player1.selectUnspentBoxes(toSpend = funds),
        outputs      = List(trueBox),
        fee          = MinTxFee,
        sendChangeTo = player1.wallet.getAddress
        )

    // We sign the transaction deposit to verify that it comes from the player 1.
    val depositTransactionSigned = player1.wallet.sign(depositTransaction)

    // We send the tx to the game blockchain.
    blockchainGame.send(depositTransactionSigned)

    // We withdraw funds from the contract 
    val withdrawBox =Box(value = maximoParaEnviar - MinTxFee,
                    script = contract(waiter.wallet.getAddress.pubKey))

    // We create the withdrawal transaction "The input will be the previous output"
    // input: depositTransactionSigned.outputs(0) 
    val withdrawTransaction = Transaction(
        inputs       = List(depositTransactionSigned.outputs(0)),
        outputs      = List(withdrawBox),
        fee          = MinTxFee,
        sendChangeTo = player1.wallet.getAddress
        )

    // Sign the transaction
    val withdrawTransactionSigned = player1.wallet.sign(withdrawTransaction)
    // We send the transaction to blockchainGame
    blockchainGame.send(withdrawTransactionSigned)
}else{
    winner = 1
    val trueBox = Box(value = maximoParaEnviar, script = trueContract)
    val depositTransaction = Transaction(
        inputs       = player2.selectUnspentBoxes(toSpend = funds),
        outputs      = List(trueBox),
        fee          = MinTxFee,
        sendChangeTo = player2.wallet.getAddress
    )
    val depositTransactionSigned = player2.wallet.sign(depositTransaction)
    blockchainGame.send(depositTransactionSigned)
    val withdrawBox =Box(value = maximoParaEnviar - MinTxFee,
                    script = contract(waiter.wallet.getAddress.pubKey))
    val withdrawTransaction = Transaction(
        inputs       = List(depositTransactionSigned.outputs(0)),
        outputs      = List(withdrawBox),
        fee          = MinTxFee,
        sendChangeTo = player2.wallet.getAddress
        )
    val withdrawTransactionSigned = player1.wallet.sign(withdrawTransaction)
    blockchainGame.send(withdrawTransactionSigned)
}

// We print the result on the screen
println("------------------------")
if (winner == 1){
    println("PLAYER 1 WIN keeps its funds.")
}else{
    println("PLAYER 2 WIN keeps its funds.")
}
println("------------------------")
println("Current balance")
println("---------------------")
player1.printUnspentAssets()
player2.printUnspentAssets()
waiter.printUnspentAssets()
println("....gas:" + MinTxFee)
println("------------------------")

```