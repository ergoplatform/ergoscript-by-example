"Heads or Tails game Contract"
=================================

* Author: ThierryM
* Created: 23/10/2021
* License: CC0
* Difficulty: (Intermediate)
* Ergo Playground Link: [Heads or Tails](https://scastie.scala-lang.org/CIhbqeisRhaDwv0SHjNB6Q)

Description
----------
This script simulates a "Heads or Tails" (flip coin) game between two players.<br />
It runs 5 games in a row, each player engaging 1 ERG for each round, the winner get all the founds minus the transaction fees.<br />
<br />
One player create the game and choose 0 or 1.<br />
The second player needs to guess the first player choice.<br />
To hide the first player choice, he creates a secret and a hash of the answer to provide the hashed result to the other player.<br />
The first player also choose the game price and the duration.<br />
The funds of the first player can be refunded after the end of the game if it did not occured.<br />
The second player is able to create the game transaction, providing his choice and the hash of the choice of the player 1.<br />
Once the transaction is signed, the first player can reaveal is choice and its secret to the second player.<br />
Both players tries to widthraw the fund only the winner can succeed.<br />
If the player 1 does not provide its secret and answer, the second player wins by default at the end of the game duration.<br />
<br />
Diagram of the game:<br />
![alt text](https://i.ibb.co/pbcpvXr/heads-or-tails-Diagram-3.png)
<br />

Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/CIhbqeisRhaDwv0SHjNB6Q)
```scala

// Heads or Tails game
//   One player create the game and choose 0 or 1.
//   The second player needs to guess the first player choice.
//   To hide the first player choice, he creates a secret and an hash to provide the hashed result to the other player.
//   The first player also choose the game price and the duration.
//   The funds of the first player can be refunded after the end of the game if it did not occured.
//   The second player is able to create the game transaction, providing his choice and the hash of the choice of the player 1.
//   Once the transaction is signed, the first player can reaveal is choice and its secret to the second player.
//   Both players tries to widthraw the fund only the winner can succeed.
//   If the player 1 does not provide its secret and answer, the player 2 wins by default at the end of the game duration.

// Required import for the Ergo Playground
import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
// import ErgoBox and hash function
import org.ergoplatform.ErgoBox
import scorex.utils.Random
import scorex.crypto.hash.{Blake2b256, Digest32}

// This is used to create a simulated blockchain (aka "Mockchain") for the Ergo Playground
val blockchainSim = newBlockChainSimulationScenario("Heads or Tails")
// In ergo contracts we work with nano ergs, each erg equals 1000000000 nanoergs 
val nanoergsInErg = 1000000000L

/////////// START GAME CONFIGURATION ///////////
// Amount to provide by each player for a game (1 ERG)
val partyPrice = nanoergsInErg
val partyPriceMinTxFee = partyPrice - MinTxFee
// Initial funds of the players (10 ERG)
val playerFunds = 10 * partyPrice
// Game period - 200mn
// After that time the deposited funds can be widthdrawed by the player without engaging the game
// If none of the player has widthdrawed the founds, the game can still be ran
val game_duration = 100
// Number of games to run in a row. More than 15/20 and scatie emulator times out.
val partyNumber = 5
///////////  END GAME CONFIGURATION  ///////////

// Define the parties: Player 1 and Player 2
val player1 = blockchainSim.newParty("Player 1")
val player2 = blockchainSim.newParty("Player 2")

// The winnerAmount is to distribute to the winner
// We need to substract the 2 transaction fees that will be paid:
//   The first player deposit the game amount (1 MinTxFee paid)
//   One of the player initiate the game transaction (MinTxFee)
//   One more transaction fee will be removed to the winner final credit for the widthrawal
val winnerAmount = 2L * partyPrice - 2L * MinTxFee
// Provide funds to the players
player1.generateUnspentBoxes(toSpend = playerFunds)
player2.generateUnspentBoxes(toSpend = playerFunds)

///////////////////////////////////////////
//  Contracts                            //
///////////////////////////////////////////
// The game contract is created by the second player using the funds from the createGameTransaction
//   The output can be spent by the second player after the end of the game, if the first player fails to provide its secret and answer for the withdrawal
//   At any time, the winner can withdraw the funds providing the answer and the secret of the first player
val gameScript = s""" 
 { // Get inputs from the createGameTransaction, this is the last box in the input list
   val p2Choice = INPUTS(INPUTS.size-1).R4[Byte].get
   val p1AnswerHash = INPUTS(INPUTS.size-1).R5[Coll[Byte]].get
   val player1Pk = INPUTS(INPUTS.size-1).R6[SigmaProp].get
   val partyPrice = INPUTS(INPUTS.size-1).R7[Long].get
   val game_end = INPUTS(INPUTS.size-1).R8[Int].get
   
   // Get the outputs register
   val p1Choice = OUTPUTS(0).R4[Byte].get
   val p1Secret = OUTPUTS(0).R5[Coll[Byte]].get
   
   // Compute the winner (the check of the correctness of the winner answer is done later)
   val p1win = ( p2Choice != p1Choice )
   
   sigmaProp (
      // After the end of the game the second player wins by default
      // This prevents the first player to block to game by not relealing its answer and secret
     (player2Pk && HEIGHT > game_end) || 
       allOf(Coll(
         // The hash of the first player answer must match
         blake2b256(p1Secret ++ Coll(p1Choice)) == p1AnswerHash,
         // The winner can withdraw
         (player1Pk && p1win )|| (player2Pk && ( p1win == false ))
       ))
   )
  }
 """.stripMargin
 
// The create game contract is created by the first player to engage the game
// It allows to cancel the game and get a refund after the end of the game
// At any time the funds can be spent by the second player if:
//   The funds are protected by the game script
//   The output value is more than twice the party price
//   The R5 register contains the hash the the answer of the first player
//   The R6 register contains public key of the first player
//   The party price and game_end are unchanged from the initial contract
val createGameScript = s""" 
  {
    val gameScriptHash = SELF.R4[Coll[Byte]].get
    val p1AnswerHash = SELF.R5[Coll[Byte]].get

    sigmaProp (
      (player1Pk && HEIGHT > game_end) ||
          allOf(Coll(
              player2Pk,
              blake2b256(OUTPUTS(0).propositionBytes) == gameScriptHash,
              OUTPUTS(0).value >= 2 * partyPrice,
              OUTPUTS(0).R5[Coll[Byte]].get == p1AnswerHash,
              OUTPUTS(0).R6[SigmaProp].get == player1Pk,
              OUTPUTS(0).R7[Long].get == partyPrice,
              OUTPUTS(0).R8[Int].get == game_end
            ))
    )
  }
 """.stripMargin

// Create the game contract providing the pubKey of player2
val gameContract = ErgoScriptCompiler.compile(Map("player2Pk" -> player2.wallet.getAddress.pubKey)
                                             , gameScript)
// Compute the hash of the script protecting the box
// It will be used to protect the deposit boxes, to force the withdrawal using the game contract
val gameHash: Digest32 = getContractScriptHash(gameContract)

// Print the state of portfolio before the game
printUnspent()

///////////////////////////////////////////
// Let's play several parties in a row   //
///////////////////////////////////////////
var a = 0L;
for(a <- 1 to partyNumber){
    runParty(a)
}

////////////// FUNCTIONS /////////////////
// runParty: Run a round of Heads or Tails between player1 and player2
//   Player 1 create the game, Player 2 finalize the game transaction, both players try to withdraw
//   partie_number: identifier of the round
def runParty( partie_number:Long ) : Unit ={
    println("------------------------------------")
    println("------------ NEW GAME "+ partie_number +" ------------")
    println("------------------------------------")
    
    // Print the state of portfolio before the game
    printUnspent()
    val game_end = blockchainSim.getHeight + game_duration
    val p1Choice:Byte = if (scala.util.Random.nextBoolean) 0x01 else 0x00
    println("Player 1 choice: " + p1Choice)
    val p1Secret:Array[Byte] = Random.randomBytes(31)
    val p1ChoiceHash: Digest32 = Blake2b256(p1Secret :+ p1Choice)
    
    // player 1 initiate the game providing the hash of his answer, game price and duration
    val createGameTransactionSigned = createGame (player1, player2, partyPriceMinTxFee, game_end, p1ChoiceHash)
  
    // The Hash of the answer, game price, duration and player1 public key are provided to player2 to create the game transaction
    // player 2 also choose between 0 and 1
    val p2Choice:Byte = if (scala.util.Random.nextBoolean) 0x01 else 0x00
    println("Player 2 choice: " + p2Choice)
    val gameTransactionSigned = playGame (createGameTransactionSigned, player2, player1, partyPriceMinTxFee, game_end, p1ChoiceHash, p2Choice)
    printUnspent()  
    
    // The second player can withdraw at any time after the end of the game
    //addToHeight(500)
    
    // player 2 tries to get the funds and fails in case he lost
    withdrawGains(player2, gameTransactionSigned.outputs(0), winnerAmount, p1Choice, p1Secret, p2Choice, partie_number)
    // At this point this player 1 reveals its choice and secret to the player 2 so he can try to withdraw
    withdrawGains(player1, gameTransactionSigned.outputs(0), winnerAmount, p1Choice, p1Secret, p2Choice, partie_number)
    withdrawGains(player2, gameTransactionSigned.outputs(0), winnerAmount, p1Choice, p1Secret, p2Choice, partie_number)

    printUnspent() 
}

// playGame: Second player executes the game contract
//   createGameTransactionSigned: Transaction that initiated the game
//   player: player who executes the game transaction (Player 2)
//   other_player: player having created the game (Player 1)
//   amount: price of the game in NanoErg
//   game_end: HEIGTH after which the game is over, player 2 wins by default when the game is over,
//             in case Player 1 does not provide its choice and the secret once the game is played
//   p1ChoiceHash: Hash of the choice of players 1 (revealed to player 2)
def playGame (createGameTransactionSigned: org.ergoplatform.ErgoLikeTransaction, 
              player:Party, 
              other_player:Party, 
              amount:Long, 
              game_end: Int,
              p1ChoiceHash: Digest32,
              p2Choice:Byte) : org.ergoplatform.ErgoLikeTransaction = {
    
    // Prepare the box for the game
    // Player2 provide:
    //         his choice (0x00 or 0x01)
    //         provide the key of the player1
    //         the party price
    //         the end of the game
    val playerBox = Box(value    = 2 * partyPrice - 2 * MinTxFee,
                         registers = Map(R4 -> p2Choice,
                                         R5 -> p1ChoiceHash,
                                         R6 -> other_player.wallet.getAddress.pubKey,
                                         R7 -> amount,
                                         R8 -> game_end),
                         script   = gameContract)

    // player creates the game transaction from his fund and the output of the initial transaction from player1
    val gameTransaction = Transaction(
        inputs       = (player.selectUnspentBoxes(toSpend = amount + MinTxFee) :+ createGameTransactionSigned.outputs(0)),
        outputs      = List(playerBox),
        fee          = MinTxFee,
        sendChangeTo = player.wallet.getAddress
    )
    val gameTransactionSigned_ = player.wallet.sign(gameTransaction)
    // Submit the transaction to the mockchain
    blockchainSim.send(gameTransactionSigned_)
    return gameTransactionSigned_
}

// createGame: First player initiate the game
//  creator: player who creates the game (Player 1)
//  other_player: the second player (Player 2)
//  amount: price of the game in NanoErg
//  game_end: HEIGTH after which the game is over, player 1 can refund if the game has not been engaged
//  p1ChoiceHash: Hash of the choice of players 1
def createGame (creator:Party, 
                other_player:Party, 
                amount:Long, 
                game_end: Int,
                p1ChoiceHash: Digest32) : org.ergoplatform.ErgoLikeTransaction = {

    // The player1 the deposit the funds providing the map of variables used
    val createGameContract = ErgoScriptCompiler.compile(Map("player1Pk"      -> creator.wallet.getAddress.pubKey,
                                                            "player2Pk"      -> other_player.wallet.getAddress.pubKey,
                                                            "partyPrice"     -> amount,
                                                            "game_end"       -> game_end
                                                            ), 
                                                        createGameScript)

    // Create the output box
    // Store the hash of the gameScript in the R4 register to ensure that the widthdrawal comes from the gameScript
    // Store the hash of the answer of creator, to be replicated in next contract
    val player1Box = Box(value     = amount,
                         registers = Map(R4 -> gameHash, 
                                         R5 -> p1ChoiceHash
                                         ),
                         script    = createGameContract)
    // Create the transaction to deposit the funds
    val createGameTransaction = Transaction(
        inputs       = creator.selectUnspentBoxes(toSpend = amount + MinTxFee),
        outputs      = List(player1Box),
        fee          = MinTxFee,
        sendChangeTo = creator.wallet.getAddress
      )
    // The player signs the trasaction and sends it to the MockChain
    val createGameTransactionSigned_ = creator.wallet.sign(createGameTransaction)
    blockchainSim.send(createGameTransactionSigned_)
    
    return createGameTransactionSigned_
}

// withdrawGains: Try to transfert the amount of gameOutput to a box owned by the player
//   player: Party doing the widthdrawal
//   gameOutput: input box from were the funds are tried to be taken
//   amount: amount of the widthdrawal (MinTxFee will be taken)
//   partie_number: party number to be printed in the message
def withdrawGains (player:Party, 
                   gameOutput:ErgoBox,
                   amount:Long,
                   p1Choice:Byte,
                   p1Secret:Array[Byte],
                   p2Choice:Byte,
                   partie_number: Long) : Unit = {

    try {
        // Create the output box to get the fund
        // Protected only by the player pubKey
        val playerWithdrawBox = Box(value  = amount - MinTxFee,
                                    registers = Map(R4 -> p1Choice, 
                                                    R5 -> p1Secret),
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

def getContractScriptHash (contract: org.ergoplatform.compiler.ErgoContract) : Digest32 = {
  // Prepare the output box for the game contract
  val outputBox = Box(value  = 1000L,
                      registers = Map(R4 -> 0x00, 
                                      R5 -> Blake2b256(Random.randomBytes(32)),
                                      R6 -> player2.wallet.getAddress.pubKey,
                                      R7 -> partyPriceMinTxFee,
                                      R8 -> 0),
                      script = contract)
  // Compute the hash of the script protecting the box
  // It will be used to protect the deposit boxes, to force the withdrawal using the game contract
  val gameHash: Digest32 = Blake2b256(outputBox.propositionBytes)
  return gameHash
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
