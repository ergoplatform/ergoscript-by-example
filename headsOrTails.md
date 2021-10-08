"Heads or Tails game Contract"
=================================

* Author: ThierryM
* Created: 10/08/2021
* License: CC0
* Difficulty: (Intermediate)
* Ergo Playground Link: [Heads or Tails](https://scastie.scala-lang.org/xsiRViZ7QFangh04gPm30g)

Description
----------
This script simulates a "Heads or Tails" (flip coin) game between two players.
It runs 5 games in a row, each player engaging 1 ERG for each round, the winner get all the founds minus the transaction fees.
A third party, Hacker, is trying to rob the funds at different stages and fails.

The funds are deposited by the players in a box that can only be spent in the game contract for a time limit.
After the time limit the players can widthdraw the funds if the game did not occur.
Once the funds are engaged in the game, only the winner can widthdraw them.
A failed widthdrawal triggers an exception signing the transaction that is catched by the code.

This example ilustrate how to protect an output box allowing to spend it only with a specific script, here our game contract.<br />
The randomness in the game contract uses a similar pattern than in the Ergo raffle contract: https://github.com/ErgoRaffle/raffle-backend/blob/master/raffle-backend/app/raffle/RaffleContract.scala.<br />
  - The output box id which is a hash is sliced (twice) and both parts are converted to int and compared.<br />
  - This hash is regenerated when the transaction is signed, allowing to choose a new winner for each game.

Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/xsiRViZ7QFangh04gPm30g)
```scala

// Heads or Tails game for 1 ERG, 5 games in a row
// For each round:
//   Two players deposit funds in a protected contract
//   Any of them can run the game by signing the game transaction
//   Both players tries to widthraw the fund only the winner can succeed
//   A third party, the "Hacker", can try to rob the funds during the game

// Required import for the Ergo Playground
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
// import ErgoBox and hash function
import org.ergoplatform.ErgoBox
import scorex.crypto.hash.{Blake2b256, Digest32}

// This is used to create a simulated blockchain (aka "Mockchain") for the Ergo Playground
val blockchainSim = newBlockChainSimulationScenario("Heads or Tails")
// In ergo contracts we work with nano ergs, each erg equals 1000000000 nanoergs 
val nanoergsInErg = 1000000000L

/////////// START GAME CONFIGURATION ///////////
// Amount to provide by each player for a game (1 ERG)
val partyPrice = nanoergsInErg
// Initial funds of the players (10 ERG)
val playerFunds = 10 * partyPrice
// Game period - 10mn
// After that time the deposited funds can be widthdrawed by the player without engaging the game
// If none of the player has widthdrawed the founds, the game can still be ran
val game_end = blockchainSim.getHeight + (5)
// Number of parties to play in a row (the scastie emulator may time out over 10/20 parties)
val partyNumber = 5
///////////  END GAME CONFIGURATION  ///////////

// Define the parties: Player 1, Player 2, and a Hacker that will ty to steal the funds
val player1 = blockchainSim.newParty("Player 1")
val player2 = blockchainSim.newParty("Player 2")
val hacker = blockchainSim.newParty("Hacker")

// The winnerAmount is to distribute to the winner
// We need to substract the 3 transaction fees that will be paid:
//   Each player deposit the game amount (2 MinTxFee paid)
//   One of the player initiate the game transaction (MinTxFee)
//   One more transaction fee will be removed to the winner final credit for the widthrawal
val winnerAmount = 2L * partyPrice - 3L * MinTxFee
// Provide funds to the players
player1.generateUnspentBoxes(toSpend = playerFunds)
player2.generateUnspentBoxes(toSpend = playerFunds)

///////////////////////////////////////////
//  Contracts                            //
///////////////////////////////////////////
// The holdScript will be used by both players to engage the funds in the game
// The player needs to provide the exact deposit amount in input
// The deposit player cannot withdraw before the end of the game (game_end)
// During the game period, both player can withdraw the found if the contract output box is protected by our gameScript following
val holdScript = s""" 
    sigmaProp ({
      allOf(Coll(
              (player1Pk && HEIGHT > game_end)|| ((player2Pk || player1Pk)  && (blake2b256(OUTPUTS(0).propositionBytes) == INPUTS(0).R4[Coll[Byte]].get)),
              INPUTS(0).value == partyPrice && (OUTPUTS(0).value == winnerAmount || player1Pk)
              ))
              })
 """.stripMargin

// The gameScript will choose a random winner and allow only him to withdraw the funds
// The same kind of randomness than in the Raffle has been used, it is based on parts of the output id (hash) converted to integer
// The winner needs to widthdraw everything
val gameScript = s""" 
 { val p1win = ( byteArrayToBigInt(SELF.id.slice(1, 8)).toBigInt < byteArrayToBigInt(SELF.id.slice(9, 15)).toBigInt )
   sigmaProp (((player1Pk && p1win )|| (player2Pk && ( p1win == false ))) && INPUTS(0).value == winnerAmount)
  }
 """.stripMargin

// Create the game contract providing the pubKey of both players, the amount for the winner
val gameContract = ErgoScriptCompiler.compile(Map("player1Pk"    -> player1.wallet.getAddress.pubKey,
                                                  "player2Pk"    -> player2.wallet.getAddress.pubKey,
                                                  "winnerAmount" -> winnerAmount)
                                             , gameScript)
// Prepare the output box for the game contract
val outputBox = Box(value  = winnerAmount,
                    script = gameContract)
// Compute the hash of the script protecting the box
// It will be used to protect the deposit boxes, to force the withdrawal using the game contract
val gameHash = Blake2b256(outputBox.propositionBytes)

///////////////////////////////////////////
// Let's play several parties in a row   //
///////////////////////////////////////////
var a = 0L;
for( a <- 1 to partyNumber){
    runParty(a)
}

////////////// FUNCTIONS /////////////////
// runParty: Run a round of Heads or Tails between player1 and player2
//   partie_number: identifier of the round
def runParty( partie_number:Long ) : Unit ={
    println("------------------------------------")
    println("------------ NEW GAME "+ partie_number +" ------------")
    println("------------------------------------")
    
    // Print the state of portfolio before the game
    printUnspent()
    
    // Players deposit the funds for the game
    val player1FundTransactionSigned = depositFund(player1,player2,partyPrice,gameHash)
    val player2FundTransactionSigned = depositFund(player2,player1,partyPrice,gameHash)
    printUnspent()
    
    // The hacker is trying to steal the funds from the Player 1 deposit before the Game starts
    withdrawGains(hacker, player1FundTransactionSigned.outputs(0), partyPrice - MinTxFee, partie_number)
    // The Player 1 is trying to steal the funds from the Player 2 deposit bot before the Game starts
    // withdrawGains(player1, player2FundTransactionSigned.outputs(0), partyPrice - MinTxFee, partie_number)
    // If the Player 1 widthdraw its own fund here, the Game transaction will fail due to INPUTS(0).value == winnerAmount
    // If the HEIGHT is increased over 5, the game is over and the refund succeed. The subsequent game would fail without the funds of player1.
    // player2 would need to do the same to get also a refund.
    // addToHeight(10)
    // withdrawGains(player1, player1FundTransactionSigned.outputs(0), partyPrice - MinTxFee, partie_number)
    
    // Create the game transaction
    // We also need to set the fee which we set to the minimum, and the amount are fixed, no change is suppoesed to be distributed
    // The two boxes in input are the output boxes of the funds deposited by the two players
    // The output box script includes the random part that will give a winner able to widthdraw all the funds
    val gameTransaction = Transaction(
        inputs       = List(player1FundTransactionSigned.outputs(0), player2FundTransactionSigned.outputs(0)),
        outputs      = List(outputBox),
        fee          = MinTxFee,
        sendChangeTo = player1.wallet.getAddress
    )
    // Flip a coin to choose who sign the transaction of the game, it does not matter
    val gameTransactionSigned = if(scala.util.Random.nextBoolean){
        player1.wallet.sign(gameTransaction)
    } else {
        player2.wallet.sign(gameTransaction)
    }
    // Submit the transaction to the mockchain and print player assets
    blockchainSim.send(gameTransactionSigned)
    printUnspent()
    // Hacker tries to get the money and fails
    withdrawGains(hacker, gameTransactionSigned.outputs(0), winnerAmount, partie_number)
    // Both players try to get the money, only the winner succeed
    withdrawGains(player1, gameTransactionSigned.outputs(0), winnerAmount, partie_number)
    withdrawGains(player2, gameTransactionSigned.outputs(0), winnerAmount, partie_number)
    printUnspent()
}

// depositFund: Create the deposit transaction for a player
//   player: player that deposit the funds
//   other_player: other party
//   amount: amount of nanoERG to be transfered (includes MinTxFee)
//   gameHash: hash of the script protecting the game output box
def depositFund( player:Party, other_player: Party, amount:Long, gameHash: Digest32 ) : org.ergoplatform.ErgoLikeTransaction = {
    // Remove the fee from deposited funds
    val partyPrice_tmp = amount - MinTxFee
    // Create the deposit contract providing the map of variables used
    val playerHoldContract = ErgoScriptCompiler.compile(Map("player1Pk"    -> player.wallet.getAddress.pubKey,
                                                            "player2Pk"    -> other_player.wallet.getAddress.pubKey,
                                                            "partyPrice"   -> partyPrice_tmp,
                                                            "game_end"     -> game_end,
                                                            "winnerAmount" -> winnerAmount
                                                      ), holdScript)
    // Create the output box
    // Store the hash of the gameScript in the R4 register to ensure that the widthdrawal comes from the gameScript
    val playerBox = Box(value    = amount - MinTxFee,
                        register = (R4 -> gameHash),
                        script   = playerHoldContract)
    // Create the transaction to deposit the funds
    val playerFundTransaction = Transaction(
        inputs       = player.selectUnspentBoxes(toSpend = amount - MinTxFee),
        outputs      = List(playerBox),
        fee          = MinTxFee,
        sendChangeTo = player.wallet.getAddress
      )
    // The player sign the trasaction and send it to the MockChain
    val playerFundTransactionSigned = player.wallet.sign(playerFundTransaction)
    blockchainSim.send(playerFundTransactionSigned)
      
    return playerFundTransactionSigned
}

// withdrawGains: Try to transfert the amount of gameOutput to a box owned by player
//   player: Party doing the widthdrawal
//   gameOutput: input box from were the funds are tried to be taken
//   amount: amount of the widthdrawal (MinTxFee will be taken)
//   partie_number: party number to be printed in the message
def withdrawGains ( player:Party, gameOutput:ErgoBox, amount:Long, partie_number: Long) : Unit = {
    try {
        // Create the output box to get the fund
        // Protected only by the player pubKey
        val playerWithdrawBox = Box(value  = amount - MinTxFee,
                                    script = contract(player.wallet.getAddress.pubKey))
        
        // Create the withdraw transaction
        // We set the input as the transaction game transaction output
        val playerWithdrawTransaction = Transaction(
            inputs       = List(gameOutput),
            outputs      = List(playerWithdrawBox),
            fee          = MinTxFee,
            sendChangeTo = player.wallet.getAddress
        )
    
        // The player sign the withdraw transaction to finalize the payment
        // Only the winner can succeed, the loser or an attacker would fail and generate an exception
        // The Exception is catched by the following catch block as it is expected
        val playerWithdrawTransactionSigned = player.wallet.sign(playerWithdrawTransaction)
    
        // Submit the withdraw transaction to the mockchain
        blockchainSim.send(playerWithdrawTransactionSigned)
        // Print the winner (Exception occures before in case of failure)
        println("------------------------------------")
        println("------- WINNER (" + partie_number + ") "+ player.name +" --------")
        println("------------------------------------")
    } catch {
        case e: java.lang.AssertionError => println("AssertionError: Failed to get the funds for " + player.name)
        case e: sigmastate.lang.exceptions.InterpreterException => println("InterpreterException: Failed to get the funds for " + player.name)
        case e: java.util.NoSuchElementException => println("NoSuchElementException: Other player already won and took the gain, you lose")
    }
}

def addToHeight( a:Int ) : Int = {
    // Simulate the time advance by increasing the HEIGHT, each unit is ~2mn
    val new_height = blockchainSim.getHeight+a
    blockchainSim.setHeight(new_height)
    println("New height: " + new_height)
    return new_height
}

def printUnspent(): Unit = {
    // Print the users wallets, which shows the nanoErg
    player1.printUnspentAssets()
    player2.printUnspentAssets()
}


```
