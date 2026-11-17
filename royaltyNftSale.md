EIP-0024 Artist Royalty NFT Sale Contract
=================================

* Author: LBCron
* Created: November 17 2026
* License: CC0
* Difficulty: Intermediate
* Ergo Playground Link: [EIP-0024 Artist Royalty NFT Sale Contract](https://scastie.scala-lang.org/p8h8aNh5R5GwCQU6cwDPuA)

Description
----------
This contract implements a decentralized, non-custodial NFT secondary-market listing box with **atomic creator royalties** aligned with [EIP-0024 (Artwork Contract / Royalty Standard)](https://github.com/ergoplatform/eips/blob/master/eip-0024.md).

When a seller lists a singleton NFT (`SELF.tokens(0)`), the listing box stores four parameters in registers `R4`–`R7`:
- **`R4[SigmaProp]` (`sellerPk`)**: The seller's public key used both to receive net sale proceeds and to authorize listing cancellation/delisting.
- **`R5[Long]` (`salePrice`)**: Gross sale price in `nanoErgs` (`salePrice > 0L`).
- **`R6[Coll[Byte]]` (`artistPropBytes`)**: The original artist/creator's `propositionBytes` that must receive the royalty share.
- **`R7[Long]` (`royaltyPermille`)**: Creator royalty rate in parts per `1000` (`0L <= royaltyPermille <= 1000L`, e.g., `50L` = `5.0%`).

### Spending Paths & Double-Satisfaction Protection
The contract allows spending via `sellerPk || sigmaProp(purchaseValid)`:
1. **Atomic Purchase Path (`purchaseValid`)**: Any buyer can spend the listing box without the seller's signature if and only if the transaction atomically:
   - Pays the seller at least `salePrice - royaltyAmount` in an output box guarded by `sellerPk.propBytes` whose **`R4[Coll[Byte]]` equals `SELF.id`**.
   - Pays the artist at least `royaltyAmount = (salePrice * royaltyPermille) / 1000L` (whenever `royaltyAmount > 0L`) in an output box guarded by `artistPropBytes` whose **`R4[Coll[Byte]]` equals `SELF.id`**.
   - Delivers the listed NFT (`SELF.tokens(0)`) to an output box with a different guarding script (`propositionBytes != SELF.propositionBytes`).
   - Binding `b.R4[Coll[Byte]].get == SELF.id` on both payout boxes prevents **Double-Satisfaction attacks** in Ergo's eUTXO model when a buyer sweeps multiple NFT listings belonging to the same seller or artist in a single transaction.
2. **Seller Cancellation Path (`sellerPk`)**: At any time before a purchase occurs, the seller can sign with `sellerPk` to delist and reclaim the NFT.



Code
----------
#### [Click Here To Run The Code Via The Ergo Playground](https://scastie.scala-lang.org/p8h8aNh5R5GwCQU6cwDPuA)
```scala

import org.ergoplatform.compiler.ErgoScalaCompiler._
import org.ergoplatform.playgroundenv.utils.ErgoScriptCompiler
import org.ergoplatform.playground._
import org.ergoplatform.Pay2SAddress

val nanoErgsInErg: Long = 1000000000L

///////////////////////////////////////////////////////////////////////////////////
// 1. EIP-0024 Artist Royalty NFT Sale Contract                                  //
///////////////////////////////////////////////////////////////////////////////////
// Register layout on the NFT Sale Listing Box (SELF):
//   - SELF.tokens(0)          : (nftTokenId, 1L) singleton NFT being sold
//   - SELF.R4[SigmaProp].get  : sellerPk (seller public key for payout & cancel)
//   - SELF.R5[Long].get       : salePrice (gross sale price in nanoErgs)
//   - SELF.R6[Coll[Byte]].get : artistPropBytes (creator propositionBytes for royalty)
//   - SELF.R7[Long].get       : royaltyPermille (royalty in parts per 1000, e.g. 50L = 5%)
//
// Both payout boxes bind R4[Coll[Byte]] == SELF.id to prevent Double-Satisfaction
// attacks if multiple listings from the same seller/artist are spent in one tx.
val royaltyNftSaleScript =
  """{
    |  val sellerPk        = SELF.R4[SigmaProp].get
    |  val salePrice       = SELF.R5[Long].get
    |  val artistPropBytes = SELF.R6[Coll[Byte]].get
    |  val royaltyPermille = SELF.R7[Long].get
    |
    |  val validRoyaltyRate = royaltyPermille >= 0L && royaltyPermille <= 1000L
    |  val royaltyAmount    = (salePrice * royaltyPermille) / 1000L
    |  val sellerAmount     = salePrice - royaltyAmount
    |
    |  val sellerBoxes = OUTPUTS.filter { (b: Box) =>
    |    b.R4[Coll[Byte]].isDefined &&
    |    b.R4[Coll[Byte]].get == SELF.id &&
    |    b.propositionBytes == sellerPk.propBytes &&
    |    b.value >= sellerAmount
    |  }
    |
    |  val artistBoxes = OUTPUTS.filter { (b: Box) =>
    |    b.R4[Coll[Byte]].isDefined &&
    |    b.R4[Coll[Byte]].get == SELF.id &&
    |    b.propositionBytes == artistPropBytes &&
    |    b.value >= royaltyAmount
    |  }
    |
    |  val nftBoxes = OUTPUTS.filter { (b: Box) =>
    |    b.tokens.size >= 1 &&
    |    b.tokens(0)._1 == SELF.tokens(0)._1 &&
    |    b.tokens(0)._2 == SELF.tokens(0)._2 &&
    |    b.propositionBytes != SELF.propositionBytes
    |  }
    |
    |  val sellerPaid   = sellerBoxes.size >= 1
    |  val artistPaid   = (royaltyAmount == 0L) || (artistBoxes.size >= 1)
    |  val nftDelivered = nftBoxes.size >= 1
    |
    |  val purchaseValid = validRoyaltyRate && salePrice > 0L && sellerPaid && artistPaid && nftDelivered
    |
    |  sellerPk || sigmaProp(purchaseValid)
    |}""".stripMargin

val listingContract = ErgoScriptCompiler.compile(Map(), royaltyNftSaleScript)
val contractAddress = Pay2SAddress(listingContract.ergoTree)
println("Royalty NFT Sale Contract P2S Address: " + contractAddress)
println("------------------------------------------------------------")

///////////////////////////////////////////////////////////////////////////////////
// 2. Scenario A: Atomic NFT Purchase with 5% Artist Royalty Split               //
///////////////////////////////////////////////////////////////////////////////////
// Create a simulated blockchain ("Mockchain") and mint a 1-of-1 artwork NFT
val blockchainSim = newBlockChainSimulationScenario("EIP-0024 Royalty NFT Sale - Purchase")
val artworkNft    = blockchainSim.newToken("ErgoGenesisArtNFT")

// Define the three market participants
val artist = blockchainSim.newParty("artist")
val seller = blockchainSim.newParty("seller")
val buyer  = blockchainSim.newParty("buyer")

// Fund seller with 2 ERG + the singleton NFT, and buyer with 110 ERG
seller.generateUnspentBoxes(toSpend = 2 * nanoErgsInErg, tokensToSpend = List(artworkNft -> 1L))
buyer.generateUnspentBoxes(toSpend = 110 * nanoErgsInErg)

val salePrice       = 100 * nanoErgsInErg // 100 ERG gross price
val royaltyPermille = 50L                 // 5.0% artist royalty (5 ERG)
val artistPropBytes = contract(artist.wallet.getAddress.pubKey).ergoTree.bytes

// Seller locks the NFT inside the Royalty NFT Sale listing box
val listingBox = Box(
  value = MinErg,
  token = (artworkNft -> 1L),
  registers = Map(
    R4 -> seller.wallet.getAddress.pubKey,
    R5 -> salePrice,
    R6 -> artistPropBytes,
    R7 -> royaltyPermille
  ),
  script = listingContract
)

val listTxSigned = seller.wallet.sign(Transaction(
  inputs       = seller.selectUnspentBoxes(toSpend = MinErg + MinTxFee, tokensToSpend = List(artworkNft -> 1L)),
  outputs      = List(listingBox),
  fee          = MinTxFee,
  sendChangeTo = seller.wallet.getAddress
))
blockchainSim.send(listTxSigned)

val onChainListingBox = listTxSigned.outputs(0)
val royaltyAmount     = (salePrice * royaltyPermille) / 1000L // 5 ERG
val sellerAmount      = salePrice - royaltyAmount            // 95 ERG

// Buyer builds the atomic purchase transaction:
//   1) 95 ERG to seller with R4 = onChainListingBox.id
//   2) 5 ERG royalty to artist with R4 = onChainListingBox.id
//   3) 1-of-1 NFT delivered to buyer
val sellerPayoutBox = Box(
  value    = sellerAmount,
  register = (R4 -> onChainListingBox.id),
  script   = contract(seller.wallet.getAddress.pubKey)
)
val artistRoyaltyBox = Box(
  value    = royaltyAmount,
  register = (R4 -> onChainListingBox.id),
  script   = contract(artist.wallet.getAddress.pubKey)
)
val buyerNftBox = Box(
  value  = MinErg,
  token  = (artworkNft -> 1L),
  script = contract(buyer.wallet.getAddress.pubKey)
)

val buyTxSigned = buyer.wallet.sign(Transaction(
  inputs       = List(onChainListingBox) ++ buyer.selectUnspentBoxes(toSpend = salePrice + MinErg + MinTxFee),
  outputs      = List(sellerPayoutBox, artistRoyaltyBox, buyerNftBox),
  fee          = MinTxFee,
  sendChangeTo = buyer.wallet.getAddress
))
blockchainSim.send(buyTxSigned)

artist.printUnspentAssets()
seller.printUnspentAssets()
buyer.printUnspentAssets()
println("------------------------------------------------------------")

///////////////////////////////////////////////////////////////////////////////////
// 3. Scenario B: Seller Delists / Cancels Listing & Reclaims NFT                //
///////////////////////////////////////////////////////////////////////////////////
val cancelSim    = newBlockChainSimulationScenario("EIP-0024 Royalty NFT Sale - Seller Cancel")
val cancelNft    = cancelSim.newToken("ErgoGenesisArtNFT_Cancel")
val cancelSeller = cancelSim.newParty("seller_cancel")
cancelSeller.generateUnspentBoxes(toSpend = 2 * nanoErgsInErg, tokensToSpend = List(cancelNft -> 1L))

val cancelListTxSigned = cancelSeller.wallet.sign(Transaction(
  inputs = cancelSeller.selectUnspentBoxes(toSpend = 2 * MinErg + MinTxFee, tokensToSpend = List(cancelNft -> 1L)),
  outputs = List(Box(
    value = 2 * MinErg,
    token = (cancelNft -> 1L),
    registers = Map(
      R4 -> cancelSeller.wallet.getAddress.pubKey,
      R5 -> salePrice,
      R6 -> artistPropBytes,
      R7 -> royaltyPermille
    ),
    script = listingContract
  )),
  fee = MinTxFee,
  sendChangeTo = cancelSeller.wallet.getAddress
))
cancelSim.send(cancelListTxSigned)

val activeCancelBox = cancelListTxSigned.outputs(0)
val cancelTxSigned = cancelSeller.wallet.sign(Transaction(
  inputs  = List(activeCancelBox),
  outputs = List(Box(
    value    = activeCancelBox.value - MinTxFee,
    token    = (cancelNft -> 1L),
    register = (R4 -> activeCancelBox.id),
    script   = contract(cancelSeller.wallet.getAddress.pubKey)
  )),
  fee = MinTxFee
))
cancelSim.send(cancelTxSigned)
cancelSeller.printUnspentAssets()

```
