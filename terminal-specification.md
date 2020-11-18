# Phonon Terminal Specification
The phonon terminal provides a user interface to interact with phonon cards. Primary functionality shall include:
1) Enable users to manage their phonon wallet (view, create, destroy phonons).
1) Enable users to connect and send/recieve phonons with another phonon card (either via a remote terminal or locally).

The following document describes the functional requirements of the terminal. Specific details about how a terminal may interact with a phonon card can be found in the [card specification](card-specification.md).

## Contents
* [1 Wallet Management](#1-wallet-management)
* [2 Sending and Receiving Phonons](#2-sending-and-receiving-phonons)

## 1 Wallet Management
At minimum, a terminal must facilitate the following management functions:
1) View owned phonons
1) Create a new phonon
1) Destroy a phonon

### View Owned Phonons
A phonon card contains a list of phonons that it posesses. The terminal can read out the list of phonon descriptors via the card interface. A phonon descriptor contains the information necessary to identify which blockchain assets are encumbered to this phonon, and the ability to query the appropriate chain and confirm validity of the asset.

It is important to note, that the phonon card is unaware of any blockchain, and thus any information contained in the descriptor is meant to aid wallet discovery, but must not be trusted finally. The appropriate blockchain must always be checked to confirm the specified assets are properly encumbered to each phonon. The structure of the phonon descriptor and the command interface to read out each descriptor can be found in the [card specification](card-specification.md).

The phonon terminal should provide the necessary UI as well as a network connection to the appropriate blockchain networks (perhaps via external block explorer) necessary to confirm validity of each asset on-chain.

### Create a New Phonon
Creating a new phonon is a 3 step process, described below:
1) Create a phonon on the card
    * The card will respond with a phonon public key
1) Convert the public key to an appropriate address format, and encumber assets to that address on the appropriate blockchain
1) Finalize phonon creation by setting the phonon descriptor on the card
    * The descriptor contains information about the asset, network, address format, etc, so that the phonon may be descoverable and spendable later from the chain. See the [card-specification](card-specification.md) for the phonon descriptor format for each supported asset.

The phonon terminal should provide the necessary UI as well as a network connection to the appropriate blockchain networks (perhaps via external block explorer) necessary to create and confirm the details of the phonon after creation.

### Destroy a Phonon
Since a phonon encumbers assets via an internal secret key, known only ever to the phonon card which currently posesses the phonon, the only way to recover the assets (aka withdraw) from the associated blockchain is to destroy the phonon and retrieve its private key. Upon retreiving the private key from the destroyed phonon, the terminal can create the appropriate blockchain transaction to transfer its assets.

A destroyed phonon can no longer be transferred to other phonon cards, but it can still be queried from the card, in case the terminal needs to retry the blockchain transaction.

The phonon terminal must provide the necessary UI as well as a network connection, necessary to select phonons to be destroyed and build transactions to transfer their asset's to an address specified by the user.

## 2 Sending and Receiving Phonons
To send or receive phonons, the terminal must provide the necessary UI to specify/confirm transaction details, as well as a connection to the reciever's card (possibly via network connection to a remote terminal). Once both parties have agreed to the transaction details, the terminal(s) must relay messages between the sender's and receiver's cards to perform the transaction.

It is important to note that phonon transactions are atomic and irreversible. Certain transactions may take on a degree of counter-party risk, if the sender does not have "exact change", the sender may have to trust the receiver to send appropriate change back; however, if the receiver never sends the change phonons, there is no recourse at the protocol level. Given this, certain transaction types may wish to take measures to limit counterparty risk, such as small incremental transfers (similar to a payment channel). Other applications may be fine with this risk, and wish to just make the transfer all at once with maximum counter party risk. We will leave handling these scenarios up to the terminal implementation.

