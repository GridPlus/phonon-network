# Phonon Card Interface Specification
The following document focuses on the card communication protocol as part of the Phonon Network. More information about the broader network is available in the [Phonon Network Specification](README.md).

## Contents
* [1 Card Usage](#1-card-usage)
* [2 Communication Protocol](#2-communication-protocol)
* [3 Command Description](#3-command-description)
  * [3.1 Command Table](#31-command-table)
  * [3.2 Data Format](#32-data-format)

## 1 Card Usage
The card is designed to support the following primitive functions for the Phonon Network: creation, storage, transfer, and destruction of phonons. A card may be attached to a trusted terminal, which will provide a user interface for users to access and use the phonons on their card.

When a card is inserted into a terminal, the card session will always follow the below procedure:
1) Open secure channel with card
1) Mutually authenticate the channel
1) Unlock card with user PIN

After the card session has been successfully started, all commands will be encrypted and decrypted with the secure channel.

Opening a secure channel involves checking that the card has a valid signed certificate from GridPlus and agreeing on secure channel keys via a Diffie-Hellman key exchange. To achieve forward secrecy and prevent command replays, the secure channel keys will rotate deterministically after each message.

The command sequence to begin a card session is illustrated below:
```
Terminal                                                                   Card
========                                                                  ======
|                                                                              |
| ::::::::::::::::::::::::::::::: OPEN_CHANNEL ::::::::::::::::::::::::::::::: |
| (clientPriv, clientPub) := gen_key()                                         |
| clientSalt := random()                                                       |
| (clientPub, clientSalt)----------------------------------------------------> |
|                                                         cardSalt := random() |
|                                         ecdhSec := ECDH(clientPub, cardPriv) |
|                                   sessionKey := sha512(clientSalt | ecdhSec) |
|                                                            aesIV := random() |
|                            channel := new_channel(encryptKey, macKey, aesIV) |
|                             sig := cardPriv.sign(sha256(sessionKey | aesIV)) |
| <----------------------------------(cardCert, cardPub, cardSalt, aesIV, sig) |
| ecdhSec := ECDH(cardPub, clientPriv)                                         |
| sessionKey := sha512(clientSalt | ecdhSec)                                   |
| GRIDPLUS_CA_KEY.verify(cardCert)                                             |
| cardPub.verify(sig, sha256(sessionKey | aesIV))                              |
| (encryptKey, macKey) := split(sessionKey)                                    |
| channel := new_channel(encryptKey, macKey, aesIV)                            |
|                                                                              |
|                                                                              |
|                                                                              |
| ::::::::::::::::::::::::::::::: MUTUAL_AUTH :::::::::::::::::::::::::::::::: |
| clientSalt := random()                                                       |
| encrypted := channel.encrypt(salt)                                           |
| (encrypted)----------------------------------------------------------------> |
|                                     clientSalt := channel.decrypt(encrypted) |
|                                                         cardSalt := random() |
|                                       encrypted := channel.encrypt(cardSalt) |
| <----------------------------------------------------------------(encrypted) |
| cardSalt := channel.decrypt(encrypted)                                       |
|                                                                              |
|                                                                              |
|                                                                              |
| :::::::::::::::::::::::::::::::: VERIFY_PIN :::::::::::::::::::::::::::::::: |
| encrypted := channel.encrypt(<USER PIN>)                                     |
| (encrypted)----------------------------------------------------------------> |
|                                            pin := channel.decrypt(encrypted) |
|                                                      success := pin.verify() |
|                                        encrypted := channel.encrypt(success) |
| <----------------------------------------------------------------(encrypted) |
| success := channel.decrypt(encrypted)                                        |
|                                                                              |
```

Card to Card Secure Channel
```
Sender Card                                   TERMINAL                              Receiver Card
========                                       ======                                =============
|                                                                                                         |
| :::::::::::::::::::::::::::::::::::::::: OPEN_CHANNEL :::::::::::::::::::::::::::::::::::::::::         |
| <---------------------------------------INIT_CARD_PAIRING                                               |
| senderSalt := random()                                                                                  |
| (senderCert, senderPub, senderSalt)-->CARD_PAIR---------------------------------------------->          |                                             
|                                                                      GRIDPLUS_CA_KEY.verify(senderCert) |
|                                                                                receiverSalt := random() |
|                                                                ecdhSec := ECDH(senderPub, receiverPriv) |
|                                                              sessionKey := sha512(senderSalt | ecdhSec) |
|                                                                                       aesIV := random() |
|                                                       channel := new_channel(encryptKey, macKey, aesIV) |
|                                            receiverSig := receiverPriv.sign(sha256(sessionKey | aesIV)) |
| <--------------------------CARD_PAIR_2<---(receiverCert, receiverPub, receiverSalt, aesIV, receiverSig) |
| GRIDPLUS_CA_KEY.verify(receiverCert)                                                                    |
| ecdhSec := ECDH(receiverPub, senderPriv)                                                                |
| sessionKey := sha512(receiverSalt | ecdhSec)                                                            |
| receiverPub.verify(receiverSig, sha256(sessionKey | aesIV))                                             |
| senderSig := senderSig.sign(sha256(sessionKey | aesIV))                                                 |  
| (encryptKey, macKey) := split(sessionKey)                                                               |       
| channel := new_channel(encryptKey, macKey, aesIV)                                                       |
| (senderSig, senderPairingIdx)-------------->FINALIZE_CARD_PAIRING-------------------------------------->|
|                                                   senderPub.verify(senderSig, sha256(sessionKey | aesIV)|
|                                  (Terminal stores channel info)<-------------------(receiverPairingIdx) |
```
## 2 Communication Protocol
Card commands are exchanged via ISO-7816 command/response APDU protocol.

TODO: Add details/links to ISO-7816 and other relevant PHY protocol resources.

## 3 Command Description
The following table contains the full list of supported commands. [Section 3.2](#32-data-format) details the data format of each command and response APDU.

### 3.1 Command Table
| Name            | CMD_ID   | Description |
|:----------------|:---------|:------------|
| [INIT](#init)                       | 0xFE | Initialize a new card by setting the pin |
| [SELECT](#select)                   | 0xA4 | Instruct the secure element to select and load the Phonon applet for use. |
| [OPEN_CHANNEL](#open_channel)       | TBD | Open a secure channel with the card. |
| [MUTUAL_AUTH](#mutual_auth)         | TBD | Mutually authenticate a newly created channel. |
| [VERIFY_PIN](#verify_pin)           | TBD | Verify the user's PIN to unlock the card for use. |
| [LIST_PHONONS](#list_phonons)       | TBD | Iterate the full list of phonons, returning 'N' phonons at a time. Optional list filters may be applied. |
| [CREATE_PHONON](#create_phonons)    | TBD | Create an empty phonon and return its public key. This phonon will be unspendable until a descriptor has been set. |
| [SET_DESCRIPTOR](#set_descriptors)  | TBD | Finalize a newly created phonon, by setting a descriptor with details about the asset it encumbers. |
| [SEND_PHONONS](#send_phonons)       | TBD | Build an encrypted transaction to transfer phonons to another card. |
| [SET_RECV_LIST](#set_recv-list)     | TBD | Optional receive whitelist, to allow a terminal to pre-approve which phonons should be accepted in a transfer. |
| [RECV_PHONONS](#recv_phonons)       | TBD | Process and receive an encrypted transaction, containing a transfer of some phonons. |
| [DESTROY_PHONONS](#destroy_phonons) | TBD | Destroy a phonon to export its private key. |

### 3.2 Data Format
The following sections describe the data format of each command and the card's response to that command.

#### INIT
* CMD: 0xFE
* P1: 0x00
* P2: 0x00

Command Data:
| Field | Length |
|:------|:-------|
| Pin | (variable)|

#### SELECT
* P1: 0x04
* P2: 0x00

Command Data:
```
+---------------+----------------+
| RID (5 bytes) | PIX (11 bytes) |
+---------------+----------------+
```
If the card is already initialized: 
Response Data:
```
+-------------------+--------------------+
| Version (3 bytes) | cardPub (variable) |
+-------------------+--------------------+
```


#### OPEN_CHANNEL
Open a secure channel with the card. All phonon operations require messages to be exchanged securely via a secure channel. Thus, this operation must be performed before any phonon operations may be performed.
> Note: After opening a channel, the terminal must immediately mutually authenticate the channel to finalize its creation.

TODO: Add data format.

#### MUTUAL_AUTH
TODO: Add data format.

#### VERIFY_PIN
* P1: 0x00
* P2: 0x00

Command Data:
```
+------------------------------+
| PIN string (variable length) |
+------------------------------+
```
Response Data
Empty. Successful response status code (0x9000) means PIN verification was successful.

#### LIST_PHONONS
TODO: Add data format.

#### CREATE_PHONON
TODO: Add data format.

#### SET_DESCRIPTOR
TODO: Add data format.

#### SEND_PHONONS
TODO: Add data format.

#### SET_RECV_LIST
TODO: Add data format.

#### RECV_PHONONS
TODO: Add data format.

#### DESTOY_PHONONS
TODO: Add data format.

