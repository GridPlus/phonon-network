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
| Name            | CMD_INS   | Description |
|:----------------|:---------|:------------|
| [SELECT](#select)                   | 0xA4 | Instruct the secure element to select and load the Phonon applet for use. |
| [IDENTIFY_CARD](#identify_card)     | 0x14 | Send a challenge salt to the card, receive back the card's identity pubkey and a signature proving it's possession of the private key  |
| [LOAD_CERT](#load_cert)             | 0x15 | Load a certificate to validate the card's public key has been signed by the Gridplus CA |
| [INIT](#init)                       | 0xFE | Initialize a new card by setting the pin |
| [PAIR](#pair)                       | 0x12 | Pair a card to a terminal by exchanging ECDH secrets and having the terminal verify a GridPlus signature on the card | 
| [OPEN_CHANNEL](#open_channel)       | 0x10 | Open a secure channel with the card. |
| [MUTUAL_AUTH](#mutual_auth)         | 0x11 | Mutually authenticate a newly created channel. |
| [VERIFY_PIN](#verify_pin)           | 0x20 | Verify the user's PIN to unlock the card for use. |
| [LIST_PHONONS](#list_phonons)       | 0x32 | Iterate the full list of phonons, returning 'N' phonons at a time. Optional list filters may be applied. |
| [GET_PHONON_PUB_KEY](#get_phonon_pub_key) | 0x33 | Fetch a phonon's public key by key index |
| [CREATE_PHONON](#create_phonon)    | 0x30 | Create an empty phonon and return its public key. This phonon will be unspendable until a descriptor has been set. |
| [SET_DESCRIPTOR](#set_descriptors)  | 0x31 | Finalize a newly created phonon, by setting a descriptor with details about the asset it encumbers. |
| [DESTROY_PHONON](#destroy_phonon) | 0x34 | Destroy a phonon to export its private key. |
| [SEND_PHONONS](#send_phonons)       | 0x35 | Build an encrypted transaction to transfer phonons to another card. |
| [SET_RECV_LIST](#set_recv-list)     | TBD | Optional receive whitelist, to allow a terminal to pre-approve which phonons should be accepted in a transfer. |
| [RECV_PHONONS](#recv_phonons)       | 0x36 | Process and receive an encrypted transaction, containing a transfer of some phonons. |


### 3.2 Data Format
The following sections describe the data format of each command and the card's response to that command.

#### INIT
* CLA: 0x80
* INS: 0xFE
* P1: 0x00
* P2: 0x00

Command Data:
| Field | Length |
|:------|:-------|
| Pin   | 6      |

Response Data: 
Empty. Successful response status code (0x9000) means PIN initialization was successful.

#### SELECT
* CLA: 00
* INS: A4
* P1: 0x04
* P2: 0x00
* LC :  8
* Data : A0 00 00 08 20 00 03 01
* LE : 00

Command Data:
| Field | Length    |
|:------|:----------|
| AppletAID | 8     |

PhononAID: {0xA0, 0x00, 0x00, 0x08, 0x20, 0x00, 0x03, 0x01}

Response changes depending on whether the card has been initialized with a pin or not. 

If Card PIN is not initialized:
Response Data TLV string:

|    Tag   |  Length  |            Value                |
|:---------|:---------|:--------------------------------|
|    0x80  | Variable | Card Secure Channel Public key  |


If the card PIN is already initialized: 
Response Data TLV string:

|    Tag   |  Length  |            Value                   |
|:---------|:---------|:-----------------------------------|
|    0xa4  | Variable | return data in TLV Data Structure  |

Return TLV Data Layout

|    Tag   |  Length  |            Value                   |
|:---------|:---------|:-----------------------------------|
|    0x8f  | 16       | Card UID                           |
|    0x80  | Variable | Card Secure Channel Public key     |
|    0x02  | 2        | Application version                |
|    0x03  | 1        | Remaining pairing slots            |
|    0x8d  | 1        | Application Capability             |

| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |


#### INIT
* CLA : 0x80
* INS: 0xFE
* P1: 0x00
* P2: 0x00
* LC: 0x26
* Data : See below

This command Initializes the PIN ( six digit ) and the Secret data ( 32 bytes) for the SecureChannel

Command Data:
| Field   |  Length  |
|:--------|:---------|
| Pin     | 6 byte   |
| Secret  | 32 bytes |

Return
| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
|  0x6A80     |  Incoming data is not 26 bytes or PIN Value is not all digits   |



#### IDENTIFY_CARD
* CLA: 0x80
* INS: 0x14
* P1: 0x00
* P2: 0x00
* LC: 0x20
* Data : 32 byte Challenge Salt
* LE : 0x00

Command Data
| Field          | Length (Bytes) | 
|:---------------|:---------------|
| Challenge Salt |    32          |


Return TLV Data Layout

|    Tag   |  Length  |            Value                   |
|:---------|:---------|:-----------------------------------|
|    0x80  | Variable | Card Public Key + Salt Signature   |


| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
|  0x6984     |  Incoming data is not SHA 256 ( 32 bytes)                       |


Response Data
| Field          | Length (Bytes) | 
|:---------------|:---------------|
| Card Identity Pub Key |  65     |
| Challenge Salt Signature |   72 - 74? | 


   
#### LOAD_CERT
TBD

#### PAIR 
* CLA: 0x80
* INS: 0x12
* P1: 0x00 for Step 1, 0x01 for Step2 
* P2: 0x00

Step 1 Command Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
| Client Salt |  32     |
| Uncompressed Pairing Pub Key |   65 | 

Step 1 Response Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
| Card Salt |  32     |
| Card Cert |    | 

Step 2 Command Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
| Cryptogram     |  32            |

Step 2 Response Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
| Pairing Index  |  1             |
| Card Salt      | 32             |

| Status word |                       Description                         |
|:------------|:----------------------------------------------------------|
|  0x9000     |  Success                                                  |
|  0x6985     |  Secure Channel is already Open                           |
|  0x6A86     |  Invalid P1 Parameter                                     |
|  0x6A80     |  Step 1 - Invalid data length                             |
|             |           Secret Key length + Public Key length  + 2      |
|             |  Step 2 - Invalid data length                             |
|             |           Looking for Secret Key length                   |
|  0x6882     |  Certificate not loaded                                   |
|  0x6982     |  Step 1 - Unable to generate Secret                       |
|             |  Step 2 - Challenge failed. Unable to verify cryptogram   |

Pair is a two step command which establishes the secrets needed for a secure channel between the terminal and a card. In step 1 the terminal sends a generate public key and salt, to which the card sends back a salt, a certificate comprised of the card's identity public key and a GridPlus CA signature on that pubkey, and a signature from that pubkey on the secret generated by combining the terminal's salt with an ecdh secret between the pair of terminal and card pubkeys. The terminal validates that signature is correct and sends back a secret hash combining the original secret with the card's salt. The terminal stores this pairing secret along with an index for the pairing slot on the card. 

#### OPEN_CHANNEL
Open a secure channel with the card. All phonon operations require messages to be exchanged securely via a secure channel. Thus, this operation must be performed before any phonon operations may be performed.
> Note: After opening a channel, the terminal must immediately mutually authenticate the channel to finalize its creation.

* CLA: 0x80
* INS: 0x10
* P1: Pairing Index
* P2: 0x00

Command Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
| Pairing Public Key  |  32       |

Response Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
|  Secure Channel Salt |  32      |
|  Init Vector    | 16            |

| Status word |                       Description                         |
|:------------|:----------------------------------------------------------|
|  0x9000     |  Success                                                  |
|  0x6A86     |  Invalid P1 Parameter. Pairing key index not valid        |
|  0x6982     |  Unable to generate Secret                                |

#### MUTUAL_AUTH
The terminal and card exchange encrypted salts and each confirm that they can successfully decrypt the message.

* CLA: 0x80
* INS: 0x11
* P1: 0x00
* P2: 0x00

Command Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
|     Salt       |       32       |

Response Data: 
| Field          | Length (Bytes) | 
|:---------------|:---------------|
|     Salt       |       32       |

| Status word |                       Description                         |
|:------------|:----------------------------------------------------------|
|  0x9000     |  Success                                                  |
|  0x6985     |  Secure Channel Encryption key not initialized            |
|  0x6881     |  Already Mutually Authenticated                           |
|  0x6882     |  Incorrect Secret length                                  |

#### VERIFY_PIN
* CLA: 0x80
* INS: 0x20
* P1: 0x00
* P2: 0x00

Command Data:
| Field          | Length (Bytes) | 
|:---------------|:---------------|
|     Pin        |       6        |

Response Data:
Empty. Successful response status code (0x9000) means PIN verification was successful.

| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
|  0x63cx     |  Pin Failed. x value is remaining tries                         |

#### CREATE_PHONON
* CLA: 0x80
* INS: 0x30
* P1: 0x00
* P2: 0x00

Command Data: None

Response Data:
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x40  | 71       | Phonon Key                             |
|    0x41  |  1       | Phonon Key Index                       |
|    0x80  | 65       | Phonon ECC Public Key Value            |

| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
|  0x6A84     |  Phonon table full                                              |

The create phonon command asks the card to generate a public key and store it in the card's list of phonons. The card returns this public key and its index to the terminal. After the terminal takes this public key and assigns a phonon to it, SET_DESCRIPTOR will be used to define additional data associated with this phonon's public key.

#### SET_DESCRIPTOR
* CLA: 0x80
* INS: 0x31
* P1: 0x00
* P2: 0x00

Command Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x50  |  15      | Phonon Descriptor                      |
|    0x41  |  2       | Phonon Key Index                       |
|    0x81  |  2       | Currency Type                          |
|    0x83  |  4       | Value                                  |                             

Currency Types Value Table: 
|   Currency         | Code    |
|:-------------------|:--------|
|  Not Set (Default) | 0x0000  |
| Bitcoin            | 0x0001  |
| Ethereum           | 0x0002  |


Response Data:
No response data.

| Status word |                      Description          |
|:------------|:------------------------------------------|
|  0x9000     |  Success                                  |
|  0x????     |  Key Not Found                            |

Set descriptor asks the card to set data for a phonon key which describes the blockchain assets that key encumbers. Once set, a descriptor cannot be modified again. 

#### LIST_PHONONS
* CLA: 0x80
* INS: 0x32
* P1: 0x00 for initial request, 0x01 to request an extended phonon list
* P2: 0x00, 0x01, 0x02 or 0x03

P1 can be set to 0x01 to denote that this is a request for the next batch of phonons from an initial list request. This should be set when list phonons returns a status indicating there are additional phonons to return which could not fit in the initial list. When P1 is set the command data will be ignored, since the card has already queued up the phonons which need to be returned in the extended list.

P2 values control how the card filter behaves, with the values switching on and off which fields the card must pay attention to when filtering.

| P2 Value | Meaning | 
|:---------|:--------|
|    0x00  | Ignore value filters |
|    0x01  | Filter on Less Than field only |
|    0x02  | Filter on Greater Than field only |
|    0x03  | Filter on Less than and Greater Than Field | 

Command Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x60  |          | Phonon Filter                          |
|    0x81  |  2       | Coin Type                              |
|    0x84  |  4       | Value Less Than or Equal to            |
|    0x85  |  4       | Value Greater Than or Equal to 


Response Data:
|    Tag   |  Length  |                   Value                              |
|:---------|:---------|:-----------------------------------------------------|
|    0x52  | variable | Phonon Collection                                    |
|    0x50  |  15      | n Phonon Descriptions (one for each returned phonon) |
|    0x83  |  4       | Phonon Value                                         |
|    0x81  |  2       | Coin Type                                            |
|    0x41  |  2       | Phonon Key Index                                     |


| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
| 0x9XXX | Success with extended list. XXX encodes the number of additional phonons left to be returned in a followup LIST_PHONONS request |

List phonons requests a collection of phonons from the card which satisfy a given filter subscription. The filter conditions are set using the P1 and P2 values along with command data describing the actual values to filter for. The card will return a list of phonon descriptions matching the filter settings. 

#### GET_PHONON_PUB_KEY
* CLA: 0x80
* INS: 0x33
* P1: 0x00
* P2: 0x00

Get phonon pub key returns the public key representing the phonon at the specified key index. This is intended to be used after filtering the list returned by list phonons, which omits the public keys in order to save space in the response. 

Command Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x41  |  2       | Phonon Key Index                       |

Response Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x43  | Variable | Transfer Phonon Packet                 |
|    0x44  |          | Phonon Complete Description  
|    0x80  | 65       | Phonon ECC Public Key Value            |

| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
| ????        |  Key Index Not Found

#### DESTROY_PHONON
* CLA: 0x80
* INS: 0x34
* P1: 0x00
* P2: 0x00

Destroy a phonon and export it's private key. 


Command Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x41  |  2       | Phonon Key Index                       |

Response Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x81  | 32       | Phonon ECC Private Key Value          |

| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
| ????        |  Key Index Not Found

#### SEND_PHONONS
* CLA: 0x80
* INS: 0x35
* P1: 0x00 for initial request, 0x01 to request an extended transfer packet
* P2: 0x00 Number of phonons request

Instructs a card to construct a packet of private phonon descriptions, consisting of a private key along with value and currency type, and encrypt it with the public key provided. If all of the requested phonons cannot fit within one response, the response will return a status value > 0x9000 to indicate how many phonons remain, and the caller must repeat the request with P1 set to 0x01 to receive the additional response data until 0x9000 is returned. 

The public key must have previously been validated as a signed GridPlus public key when the card to card secure channel was established. 

The Phonon Key Index List is an array of 2 byte key indices. 

Command Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x42  | N * 2    | Phonon Key Index List                  |


Response Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x43  | N * 46   | Phonon Transfer Packet                 |
|    0x44  |  44      | N Phonon Private Descriptions          |
|    0x81  |  32      | Phonon ECC Private Key Value           |
|    0x83  |  4       | Phonon Value                           |
|    0x81  |  2       | Coin Type                              |


| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |
| 0x9XXX | Success with extended list. XXX encodes the number of additional phonons left to be returned in a followup LIST_PHONONS request |


#### SET_RECV_LIST
TODO: Add data format.

#### RECV_PHONONS
* CLA: 0x80
* INS: 0x36
* P1: 0x00
* P2: 0x00

Counterpart to SEND_PHONONS, this command instructs a card to receive a phonon transfer packet received from another card's SEND_PHONONS response. This packet is encrypted to the receiving card's public key by a previous card to card secure channel request. The receiving card decrypts the packet and stores the contained phonons along with their descriptions. 

Command Data: 
|    Tag   |  Length  |            Value                       |
|:---------|:---------|:---------------------------------------|
|    0x43  | N * 46   | Phonon Transfer Packet                 |
|    0x44  |  44      | N Phonon Private Descriptions          |
|    0x81  |  32      | Phonon ECC Private Key Value           |
|    0x83  |  4       | Phonon Value                           |
|    0x81  |  2       | Coin Type                              |


Response Data: 
No Response Data, just Status Code


| Status word |                      Description                                |
|:------------|:----------------------------------------------------------------|
|  0x9000     |  Success                                                        |



