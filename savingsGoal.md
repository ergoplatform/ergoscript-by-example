# Savings Goal Contract

- Author: realfakeprogrammer
- Created: 7/19/2022
- License: CC0
- Difficulty: Beginner
- Ergo Playground Link: [Savings Goal Contract](https://scastie.scala-lang.org/LsXG9ehaQHi1dzPL2VRjaw)

## Description

This is a simple smart contract that allows a user to save funds up to an amount that they choose. Once their savings goal is reached, the user is able to withdraw the funds.

This contract demonstrates how to lock funds up to set amount before withdraw

The contract also demonstrates how to take multiple boxes as inputs and output a single box

## Code

[Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/LsXG9ehaQHi1dzPL2VRjaw)

    import org.ergoplatform.compiler.ErgoScalaCompiler._
    import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
    import org.ergoplatform.playground._
    
    // The contract below allows a user to send funds to a contract and then withdraw the funds once a "goal" amount has been reached
    // The following expression in the contract (SELF.value >= goal || OUTPUTS(0).value > INPUTS(0).value) checks 2 conditions
    // The first part -- SELF.value >= goal -- checks if the value of the contract is greater than or equal to the goal set
    // If the value is greater than or equal to the goal set then the box can be spent (allowing the funds to be withdrawn)
    // The second part -- OUTPUTS(0).value > INPUTS(0).value -- checks if the new value of the contract is greater than the old value
    // If the new value is greater than the old value the box can be spent (allowing for an additional deposit to be made)
    // The "||" seperating the 2 condtions is an operator the checks if either the first condition OR the second condition evaluates to true
    // The second part of the expression -- && saverPK -- ensures that only the saver can withdraw or deposit funds to the contract

    val savingsGoalScript = s"""
        {
        sigmaProp((SELF.value >= goal || OUTPUTS(0).value > INPUTS(0).value) && saverPK)}
        """.stripMargin

    ///////////////////////////////////////////////////////////////////////////////////
    // Prepare A Test Scenario //
    ///////////////////////////////////////////////////////////////////////////////////

    // Create a simulated blockchain (aka "Mockchain")
    val blockchainSim = newBlockChainSimulationScenario("Savings Goal Scenario")

    // Define a saver
    val saverParty = blockchainSim.newParty("saver")

    // Define initial amount of nanoErgs in saver's wallet
    val saverFunds = 100000000000L

    // Create a Savings Goal Contract which specifies the "goal" amount of the contract
    val savingsGoalContract = ErgoScriptCompiler.compile(Map("goal" -> 10000000000L,
                                                            "saverPK" -> saverParty.wallet.getAddress.pubKey), savingsGoalScript)

    // Generate inital saverFunds in saver's wallet
    saverParty.generateUnspentBoxes(toSpend = saverFunds)
    saverParty.printUnspentAssets()
    println("----------- Saver's Initial Funds -----------")

    ///////////////////////////////////////////////////////////////////////////////////
    // Deposit Funds Into Savings Goal Contract //
    ///////////////////////////////////////////////////////////////////////////////////

    // Create an output box with the saver's deposit locked under the contract
    val firstDeposit:Long = 6000000000L

    val savingsGoalBox = Box(
        value = firstDeposit,
        script = savingsGoalContract,
    )

    // Create first deposit transaction which locks the saver's deposit under the contract
    val depositTransaction = Transaction(
        inputs       = saverParty.selectUnspentBoxes(toSpend = firstDeposit),
        outputs      = List(savingsGoalBox),
        fee          = MinTxFee,
        sendChangeTo = saverParty.wallet.getAddress
        )

    // Print depositTransaction
    println(depositTransaction)

    // Sign the depositTransaction
    val depositTransactionSigned = saverParty.wallet.sign(depositTransaction)

    // Submit tx to blockchain simulation
    blockchainSim.send(depositTransactionSigned)
    saverParty.printUnspentAssets()
    println("----------- First Deposit Complete -----------")
    
    // Create second deposit, locking more of the saver's funds under the contract
    val secondDeposit:Long = 4000000000L

    // Create second output box which locks the new total under the contract
    val secondsavingsGoalBox = Box(
        value = firstDeposit + secondDeposit,
        script = savingsGoalContract
    )

    // Create second deposit transaction which combines the saver's two deposits under the contract
    // Inputs are the new deposit and the box containing the initial deposit
    // Output is a new box with the combined value 
    val secondDepositTransaction = Transaction(
        inputs       = List(depositTransactionSigned.outputs(0)) ++ saverParty.selectUnspentBoxes(toSpend = secondDeposit),
        outputs      = List(secondsavingsGoalBox),
        fee          = MinTxFee,
        sendChangeTo = saverParty.wallet.getAddress
        )

    // Print secondDepositTransaction
    println(secondDepositTransaction)

    // Sign the secondDepositTransaction
    val secondDepositTransactionSigned = saverParty.wallet.sign(secondDepositTransaction)

    // Submit tx to blockchain simulation
    blockchainSim.send(secondDepositTransactionSigned)
    saverParty.printUnspentAssets()
    println("----------- Second Deposit Complete -----------")
    
    ///////////////////////////////////////////////////////////////////////////////////
    // Withdraw Funds Locked Under Savings Goal Contract //
    ///////////////////////////////////////////////////////////////////////////////////
    // Reduce One Of Above Deposit Values To Simulate A Premature Withdraw //
    // Script Reduces To False When Withdraw Transaction Is Signed //
    ///////////////////////////////////////////////////////////////////////////////////

    // Create an output box which withdraws the funds to the saver
    // Subtracts MinTxFee from value to account for tx fee
    val withdrawBox      = Box(value = firstDeposit + secondDeposit - MinTxFee,
                            script = contract(saverParty.wallet.getAddress.pubKey)
                            )

    // Create the withdrawTransaction
    val withdrawTransaction = Transaction(
        inputs       = List(secondDepositTransactionSigned.outputs(0)),
        outputs      = List(withdrawBox),
        fee          = MinTxFee,
        sendChangeTo = saverParty.wallet.getAddress
        )

    // Print withdrawTransaction
    println(withdrawTransaction)

    // Sign the withdrawTransaction
    val withdrawTransactionSigned = saverParty.wallet.sign(withdrawTransaction)

    // Submit tx to blockchain simulation
    blockchainSim.send(withdrawTransactionSigned)

    // Print the saver's wallet, which shows that the coins have been withdrawn (with same total as initial, minus the MinTxFee * number of txs, in this case, 3 txs)
    saverParty.printUnspentAssets()
    println("----------- Withdraw Complete -----------")

