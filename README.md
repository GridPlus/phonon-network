# Phonon Network Specification
## Contents
* [1 Network Overview](#1-network-overview)
  * [1.1 What is a phonon?](#11-what-is-a-phonon)
* [2 Phonon Card](#2-phonon-card)
  * [2.1 Creating Phonons](#21-creating-phonons)
  * [2.2 Transferring Phonons](#22-transferring-phonons)
  * [2.3 Destroying Phonons](#23-destroying-phonons)
* [3 Phonon Terminal](#3-phonon-terminal)
  * [3.1 Transfer Backends ](#31-transfer-backends)
  * [3.2 Asset Verification](#32-asset-verification)
  * [3.3 Supported Assets](#33-supported-assets)
  * [3.4 Viewing Card Balance](#34-viewing-card-balance)
  * [3.5 Transaction Building](#35-transaction-building)

## 1 Network Overview
The Phonon Network is a peer-to-peer network independent of the internet. It leverages the hardware security of secure-elements/HSMs (smart cards) to allow the secure storage and transfer of "phonons". Transactions occur without the need to broadcast it to an external ledger, and are fully contained between the two parties involved.

### 1.1 What is a Phonon?
A phonon is a representation of a public/private key pair and some information (phonon descriptor) describing a digital asset encumbered to that key pair (e.g. a Bitcoin UTXO, or an Ethereum or ERC20 token). In this way, exchanging phonons is a secure mechanism of exchanging private keys, and consequently exchanging ownership of a digital asset without ever broadcasting a transaction to the asset's associated network/blockchain.

Traditionally, exchanging private keys would expose a user to significant counterparty risk; the recipient has no way of knowing that the sender didn't keep a copy of the private key to steal the assets back, etc. The Phonon Network solves this problem, by ensuring private keys are never known publicly, and are only transferred between cards which can be trusted to atomically delete its copy of a phonon after it has been transferred. In this way, when transferring between phonon cards, the receiver can be confident that the received private key is only known to his card, and therefore the digital assets encumbered to his phonon's key are solely his. To ensure phonons are not duplicated, cards will only transact with other official GridPlus cards, which can be trusted to abide by the network rules. Accordingly, each card first checks the other card possesses a certificate signed by GridPlus. Communication between cards is encrypted using the cards' unique keys, to prevent transactions from being broadcast to multiple cards.

This functionality relies on the following key principles:
1) A phonon private key must be known only to a trusted phonon card (until withdrawal from the network).
1) After sending a phonon or withdrawing a phonon from a card, that card will destroy its record of the phonon.

Interaction with the Phonon Network involves two key technologies:
1) [Phonon Card](#2-phonon-card) - A secure environment that executes secure atomic phonon transfers.
1) [Phonon Terminal](#3-phonon-terminal) - A card terminal to provide a user interface & facilitate transfers between phonon cards.

## 2 Phonon Card
A phonon card is a secure environment to store and transfer phonons. There are three primary ways to interact with the Phonon Network:
* create phonons by locking digital assets to a phonon key (e.g. on a blockchain)
* transfer phonons between cards
* destroy phonons (export phonon private key to unlock its digital assets)

Each of these three interactions are described in further detail below. A detailed description of the card command interface can be found in the [card specification](card-specification.md).

### 2.1 Creating Phonons
The private key for each phonon must only be known to the card. As such, to create a phonon, a user must ask the card for a public key to a new phonon. The user can then use that public key to commit assets on a blockchain to that phonon (e.g. create Bitcoin address from pubkey and send some BTC to that address). The user must also provide the card some information about the asset that the phonon represents. In the future, when it comes time to withdraw a phonon, this information will give the user enough information to redeem the associated digital asset (e.g. which blockchain & address format this phonon asset exists on).

The creation process goes as follows:
1) User requests a new phonon public key from card.
1) User deposits digital assets on a blockchain to an address corresponding to this public key.
1) User finalizes phonon creation, by providing info about the associated digital asset (blockchain network, address format, etc).

### 2.2 Transferring Phonons
Phonons may only be transferred to/from another authentic phonon card. This prevents exchanging with malicious cards that may duplicate or leak phonon private keys. This is the key to preventing double-spending on the network. To achieve this, each phonon card is provided with a certificate signed by GridPlus. Before transferring phonons, a card must first check that the other party has a valid signed certificate, and possesses the private key that was certified. Then, a transaction can be built and encrypted using a shared secret only known to those two cards (via ECDH key exchange). When the sender's card emits the transaction, the sender's card automatically deletes the phonons internally, so that it will not be able to spend it twice. At this point, the encrypted transaction should now be treated as a collection of phonons. The receiver's card (and only that card!) can import and receive that phonon transaction. Consequently, if the transaction is lost or the receiver's card is lost, the phonons spent in that transaction will also be lost.

There is one more detail necessary to prevent replay attacks. As described up to this point, a phonon transaction could theoretically be replayed `N` times to the receiver's card, tricking the card into receiving the phonons `N` times. To prevent this, we require that each transaction correspond to an invoice ID from the receiver. This ensures that replayed transactions will be detected and discarded, as they all refer to the same invoice ID. In practice, this adds negligible complexity, because this extra invoice ID can be sent with the receiver certificate at the beginning of the transaction. This invoice ID is simply a unique nonce, and requires no information about the details of the transaction.

Lastly, to prevent certain attacks where an attacker may wish to DoS your card with bogus phonons, the card will provide the ability to let a terminal build a whitelist of phonons that may be accepted by the card. With this approach, a terminal will negotiate the details of a transfer between two cards (and check the sender's phonon public keys on chain for validity, etc). Once the transfer details are confirmed, the terminal can provide a whitelist of phonons to accept. This ensures the card will only accept phonons the terminal has already deemed valid, thus preventing the need for a lengthy interactive process with the card to detect and remove bogus phonons.

### 2.3 Destroying Phonons
Eventually a user may wish to unlock & redeem the digital assets associated with a phonon. In this case, the user can specify a number of phonons to be exported from the card. The card will respond by exporting each requested phonon (private key and asset data), so that the user can build the appropriate blockchain transactions to redeem the associated digital assets. Importantly, the act of exporting a phonon's private key _destroys_ that phonon. That phonon will be deleted from the card, and no longer transferable on the Phonon Network.

## 3 Phonon Terminal
A phonon terminal provides a physical interface to a phonon card, as well as communication backend(s) between cards to facilitate phonon transfers. Transfers may be facilitated via any number of communication backends between two cards. See [Transfer Backends](#31-transfer-backends) for a list of supported backends.

When receiving a phonon, it is important for the receiver to check that the phonon actually has assets locked up on the appropriate blockchain. For this reason, a terminal must also provide the ability to read the phonons on a card and check the value of those phonons on the appropriate network. See [Supported Assets](#32-supported-assets) for more information about supported asset types.

See the [terminal specification](terminal-specification.md) for more detail about the functional design requirements of a phonon terminal.

### 3.1 Transfer Backends
At this time, we specify the following transfer backends a terminal may choose to implement. More backends may be added in the future (brownie points to anyone who specs out a sneakernet remote backend :beers:):
1) Local transfer between one external card and one card embedded internally to the terminal.
1) Remote transfer between a local card and a remote card, via network connection to a remote terminal.

A phonon terminal must provide the ability to transfer phonons between cards via at least one of the specified backends.

### 3.2 Asset Verification
A phonon stores the private key with an associated public/private keypair. It is intended that this keypair is used to encumber assets on an external network/blockchain. A phonon descriptor is then programmed to the phonon describing which assets are encumbered and how. This allows a phonon to be transferred without broadcasting a transaction to the blockchain for every transfer.

This encumbrance, however, depends entirely on the consensus of that external network. As such, it is up to the terminal to check that the encumbrance exists as expected. For example, a user may create a phonon and encumber BTC, ETH, or any other digital asset (including hard forks like ETC or BCH) with the phonon's public key. It is up to the terminal to read the public key and phonon descriptor of each phonon and check the appropriate network to confirm the assets are indeed encumbered to the phonon's public key.

### 3.3 Supported Assets
A phonon may encumber many different types of assets. To create, verify, or destroy a phonon, a terminal must know how to interpret the type of asset encumbered by that phonon. Below is a list of supported assets and the appropriate phonon descriptors for each asset type:

| Asset    | Descriptor Information |
|:---------|:-----------------------|
| Bitcoin  | Network (mainnet, testnet, etc), address format (legacy, segwit, etc) |
| Ethereum | Network (mainnet, testnet, etc), account type (user, contract), optional contract address, optional contract data |

> See the [card specification](card-specification.md) for a detailed specification of the phonon structure, data formats, and card commands to create/transfer/destroy a phonon.

### 3.4 Viewing Card Balance
A terminal should provide the option to view a card's balance. This is a two step process. First, the terminal should request phonon descriptors from the card, to discover every phonon possessed by the card. Second, the terminal should check the appropriate blockchain(s) for validity of each phonon asset, as described in each phonon's descriptor.

> Caution: The information in a phonon descriptor must be externally validated on the appropriate blockchain(s)! The card has no knowledge of external networks, and thus cannot guarantee validity of a phonon's assets.

### 3.5 Transaction Building
A terminal provides the ability for two cards to transact with each other via one of the transfer backends described above. The user may interact directly with the terminal to initiate a transaction, or the terminal may attempt to automatically initiate a transaction (e.g. in a point of sale scenario). We will leave these details to the terminal implementation, and focus solely on the construction of a transaction.

Note, that a transaction may involve only one terminal (i.e. a local transfer) or two terminals (i.e. a remote transfer). Remote transfers are an extension of local transfers, in which the transaction structure is identical, but the terminals require an extra step(s) to negotiate the details of the transaction via a remote connection. The details of a remote connection are application dependent, and we wish to focus solely on the construction of a transaction. Thus, this section will be described in the context of a local transfer only.

To build a transaction, the terminal must perform the following procedure:
1) Discover relevant phonons on the sender's card.
1) Discover relevant phonons on the receiver's card.
1) Check all phonons for validity on the appropriate blockchain(s).
1) Select the phonons the sender should send.
1) (If applicable) Select phonons the receiver should send back as change.
1) Instruct the sender's card to build a transaction to send the selected phonons.
1) (If applicable) Instruct the receiver's card to build a transaction to send the selected change phonons.
1) Provide the receiver's card with the encrypted transaction from the sender.
1) (If applicable) Provide the sender's card with the encrypted change transaction from the receiver.

> Caution: This transaction process may expose one or both parties to counterparty risk (e.g. if the receiver removes his card before transmitting the change transfer). At minimum, users should be made aware of their counterparty risk. A smart terminal may even take steps to reduce this risk, by breaking the transaction into smaller pieces, or requiring both cards to send their half of the transaction before receiving the other half (although, this has a different set of tradeoffs to be considered). These decisions are application dependent, and thus we leave these details to the terminal implementers.

The process of discovering which phonons exist on each card may be time consuming. The card interface is comparatively slow and a card may have many phonons to select from. Given this, the cards will provide a filter mechanism, by which the terminal may request a subset of phonons that satisfy a certain criteria. E.g. if Alice wishes to send BTC to Bob, the terminal may request only phonons representing BTC from each card. The filter mechanism is described in further detail in the card specification. Note, a card's filtering capabilities depend solely on information available in each phonon descriptor. It is possible that the information in the descriptor is not valid on the appropriate blockchain due to external factors (re-orgs/forks/etc). Thus, the card filters should be treated as an untrusted convenience tool and the final validity of each phonon must still be checked on the appropriate blockchain(s).

> Caution: The information in a phonon descriptor must be externally validated on the appropriate blockchain(s)! The card has no knowledge of external networks, and thus cannot guarantee validity of a phonon's assets. 

