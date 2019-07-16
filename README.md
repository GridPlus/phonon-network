# Phonon Network Specification

The Phonon Network is a layer 2 scaling solution for public blockchain networks. It is primarily designed to function on the Ethereum network, but can also be used on Bitcoin and other UTXO-based chains. Phonon can be thought of as a L2 scaling technique where transactions are completely validated between participants of a direct transaction. Nothing is shared with users on the Phonon network outside of the participant set for a given transaction. This yields significant benefits to both scalability and privacy relative to large, highly decentralized blockchain networks.

To achieve this design topology, Phonon uses **hardware enforced security** to prevent double spend attacks, specifically via smart cards that cannot be re-programmed or copied. Although the Phonon Network is theoretically card-agnostic, it is designed to be used with SafeCards, which have a specific Java card applet (see [here](https://github.com/GridPlus/safe-card)).

The security model of a Phonon-compatible smart card derives from:

1. The card's identity, which is derived from its internal physically unclonable function (PUF). A PUF is a digital fingerprint derived from physical imperfections in the manufacturing process of a secure enclave chip.
2. Certification of each card's identity by the card issuer. This comes in the form of an ECDSA signature of the card's identity public key by a known card issuer's signing key.

### What is a Phonon?

"Phonons" are discrete packets of value which may be transmitted across the Phonon network. They are **created by on-chain deposits**, which are associated with a particular public key (a derivative of the recipient card's identity public key - more on this later). Each deposit may contain one or more non-fungible phonons, which are similar to the concept of a "bill" (i.e. has a specific denomination and cannot be divided). 

### How are Phonons Transferred?

Each phonon contains an amount, an asset type, a private key, and some other metadata. Because private keys are being passed, all phonon transmission happens over secure (encrypted) channels between counterparty cards. The data is encrypted/decrypted using a shared AES secret which is the ECDH shared secret of the cards' identity key pairs.

### How do Withdrawals Work?

When a user wishes to withdraw a phonon onto the blockchain network from which it originated, they call a function on the card, which generates the relevant payload and makes a signature needed to satisfy the withdrawal conditions (these vary depending on the network and/or withdrawal implementation). *The signature originates from the **private key contained in the phonon itself** - once the withdrawal occurs, the phonon (including this key) is deleted from the card.* At this point, it is up to the user to package this signature into a transaction and make the withdrawal on-chain.

> Note that the data being signed is passed to the card. In the case of Ethereum, this is a proof that would be sent to a smart contract. For Bitcoin and similar chains, it would be the script needed to unlock the coins on-chain.

### Where is the Trust?

Trust in the Phonon network comes down to two things:

1. The hardware is correct (trust in manufacturer)
2. The software is correct (trust in issuer)

Every issuer must assume 1 to be true at all times, thus all trust is pushed to the issuer.

#### Issuing and Signing Cards

When a card is issued, a unique identity key is generated when the software is installed. The identity public key is signed by the issuer (using an ECDSA signature) and this validating signature is saved on the card in such a way that it cannot be changed or removed. It is also published on an accesible online registry (e.g. an Ethereum smart contract) such that any participant can verify that a counterparty's card's identity public key was signed by a recognized card issuer.

End each card issuer's public key must be itself signed by a **meta issuer**, which is likely to be an n-of-m multisignature scheme. The meta issuer publishes its *own* registry of *issuer public keys*.

Thus, there are *two* online registries:

1. A registry containing card identity public keys signed by their respective card issuers' keys
2. A registry containing a set of "approved" card issuers' oublic keys, signed by a "meta issuer" key

#### Validating Signatures During Transfer

Each card is loaded with the public key of the meta issuer, which is used to prove the root of trust when receiving a phonon (explained in detail in a later section). Roughly, the check is as follows:

1. Alice wants to send Bob a phonon
2. Alice encrypts the phonon payload with a shared secret derived from her identity private key and a public key Bob sent her
3. Alice sends the phonon payload as well as her card's identity public key, the signature of her card's id public key by her card issuer, and the signature of her card issuer's public key by the meta issuer
4. Bob receives and decrypts the payload, verifying that Alice owns the private key corresponding to what she claims is her id public key (otherwise the shared secret would be different and the phonon would not be properly decrypted, in which case this process is moot)
5. Bob's card validates the first signature (i.e. that the purported card issuer signed Alice's id public key)
6. Bob's card validates the second signature (i.e. that Alice's card's issuer was signed by the meta issuer)
7. Bob's card saves the phonon. Success!

In order to do this validation, Bob's card must have a copy of the meta issuer's public key - this allows it to establish the root of trust. After establishing the root, he further establishes the chain of trust, i.e. that the validated card issuer validated Alice's card. This gives him confidence that Alice's card is a valid card within the Phonon Network.

# Structure of a Phonon

Each phonon represents a non-fungible "packet" of tokens. It contains the following minimum set of data, which may be stored in various ways (covered in later sections):

1. **Receiving private key:** at deposit time, each phonon must be sent to a different address. The deposit address corresponds to a private key, which is passed between participants in the network as the phonon is spent. So long as private keys aren't reused, any counterparty risk (which should be obviated by hardware-enforced rules, but ya know... crypto people) is constrained to the individual phonon
2. **Network ID:** an identifier for the network from which this phonon derives. A recipient of a phonon should be careful to ensure this id corresponds to the network on which he expects the tokens to exist. In the case of Ethereum, this network id maps to a settlement smart contract address (20 bytes)
3. **Asset ID:** an identifier for the asset describing this phonon. For Ethereum, a list of accepted asset ids is stored on-chain; these correspond to a token address and, for NFTs, a token id. For Bitcoin, there is only one asset id (`0`).
4. **Tx Data:** usually a transaction hash and, in the case of Bitcoin, a UTXO index in that transaction.
4. **Amount:** the number of tokens (in atomic units).

# Provisioning a Card

Phonon is designed to work with [Java cards](https://www.oracle.com/technetwork/java/embedded/javacard/overview/index.html). To initialize a Java card, you first need a card reader (e.g. [HID OMNIKEY](https://www.hidglobal.com/products/readers/omnikey) series). While many variants exist, the Phonon Network is designed to *prefer* a secure interface, which exists in the GridPlus [Lattice1](https://gridplus.io/lattice).

> NOTE: The following code snippets (and most card-based code/pseudocode in this document) is based on the [GridPlus SafeCard Applet](https://github.com/GridPlus/safe-card), which is itself a fork of the [Status Keycard Applet](https://github.com/status-im/status-keycard). The Phonon specification is applet-agnostic, but these applets should serve as good starting points for prospective Phonon implementors.

## Installing and Initializing the Applet

Part of the manufacturing/issuing process involves installation and initialization of the Java card applet. Each step occurs only once - if someone attempts to reinstall an applet, the card will lose its identity keystore. 

First, the card applet source code (e.g. the SafeCard applet) is compiled and flashed onto the card using the reader. This is called **installation**.

After installing the applet, the issuer calls the `init()` function, which looks something like this:

```
public KeycardApplet(byte[] bArray, short bOffset, byte bLength) {
    ...setup...
    
    // Create and configure identity keypair
    idPublic = (ECPublicKey) KeyBuilder.buildKey(KeyBuilder.TYPE_EC_FP_PUBLIC, SECP256k1.SECP256K1_KEY_SIZE, false);
    idPrivate = (ECPrivateKey) KeyBuilder.buildKey(KeyBuilder.TYPE_EC_FP_PRIVATE, SECP256k1.SECP256K1_KEY_SIZE, false);
    secp256k1.setCurveParameters(idPublic);
    secp256k1.setCurveParameters(idPrivate);

    // Load the identity keypair with data
    byte[] privBuf = new byte[Crypto.KEY_SECRET_SIZE];
    crypto.random.generateData(privBuf, (short) 0, Crypto.KEY_SECRET_SIZE);
    idPrivate.setS(privBuf, (short) 0, Crypto.KEY_SECRET_SIZE);
    byte[] pubBuf = new byte[Crypto.KEY_PUB_SIZE];
    secp256k1.derivePublicKey(privBuf, (short) 0, pubBuf, (short) 0);
    idPublic.setW(pubBuf, (short) 0, Crypto.KEY_PUB_SIZE);

    ...other stuff...
  }
```

Here we generate an "identity" key pair(`idPublic`/`idPrivate`), which is used to identify the card. `crypto.random.generateData` uses the card's PUF (entropy) to create a random private key, which fills in the key pair objects.

At this point, the card is **initialized**. While other key pairs (related to holding crypto assets) may be added later, the only one needed for Phonon is the identity key, which cannot be changed once it is initialized.

## Proving Identity

The card's identity is captured in `idPublic` above. Any user with a card reader may request this public key at any time after the card is initialized.

The card may "prove" its identity to any requester with a card reader using a challenge/response mechanism:

```
private void authenticate(APDU apdu) {
    ...
    
    // Copy the input hash into a buffer
    byte[] msgHash = new byte[Crypto.KEY_SECRET_SIZE];
    Util.arrayCopyNonAtomic(apduBuffer, (short) ISO7816.OFFSET_CDATA, msgHash, (short) 0, Crypto.KEY_SECRET_SIZE);

    ...
    
    // Add signature of msg hash
    Signature tmpSig = Signature.getInstance(Signature.ALG_ECDSA_SHA_256, false);
    tmpSig.init(idPrivate, Signature.MODE_SIGN);
    tmpSig.signPreComputedHash(msgHash, (short) 0, MessageDigest.LENGTH_SHA_256, apduBuffer, sigOff);
   
    ...
  }
```

Here the card receives a hash and signs it using its identity key (`idPrivate`), returning that signature as a response. Thus, the requester can verify that the public key yeilded from the card earlier corresponds to the private key which made this signature.

## Certifying a Card

Each SafeCard is loaded with a certificate from the card issuer. This is nothing more than a signature on the card's identity public key (which the card has proven to have) by an ECDSA key pair controlled by the issuer at the time of card issuance. Once loaded, the cert cannot be modified or removed.

The cert can be inspected at any time by any user with a card reader. It proves the identity of the card issuer and, in the case of GridPlus, is a prerequisite for most SafeCard-based interactions on the Lattice1 (many are independent to the Phonon Network).

### Trusted Certificate Signers

When the certificate is loaded, its signer (whose public key is also included in the payload) is also verified to have signed the certificate. As long as this check passes, the certificate loading should succeed. The signer's public key is stored globally to validate interactions with other cards later on.

> Unfortunately, cards cannot interoperate with unknown certificate signers, as this would undermine the entire security model of the system. We are still researching how we might, in the future, interoperate with other card issuers should they wish to use the Phonon Network.

# Accounting on the Card

Data is stored both on the public blockchain (see: *On-Chain Settlements*) and on the card. This section covers data types, deposits, and withdrawals on Phonon-compatable cards.

## Getting a Deposit Address

Phonons are not deposited to the card's identity address, but rather to an address derived from the identity key:

```
byte[] getDepositKey(long nonce) {
    byte[] toReturn = new byte[PRIVATE_KEY_LEN];
    // Hash idPrivate with provided nonce to get a new private key
    Util.arrayCopyNonAtomic(toReturn, 0, sha256(idPrivate, nonce), PRIVATE_KEY_LEN);
    return toReturn;
}
```

The derivation is simply `sha256(nonce, idPrivate)`.

### Nonce Tracking

Each deposit key is the hash of a **nonce** with the identity key. A global nonce is kept on each card; every time a phonon is deposited, the global nonce is incremented. More on this later. 

> Note that depositing is not the same thing as generating a deposit address!

## Storing Phonons

Phonon storage is described on the card as:

```
class Phonon {
  public byte[32] owner;     // Private key that can spend or withdraw the phonon
  public byte assetId;       // Type of asset being transferred (e.g. ETH, BTC, ERC20 X)
  public short amount;       // uint16 representing the asset value
  public byte decimals;      // ETH only: (amount * 10^decimals) represents the value in atomic units
  public byte[33] extraData; // Data depending on the network
  public byte networkId;     // Index (on-card) on which to determine which network these tokens exist on
}

private Phonon[] phonons;
```

#### `owner`

The private key associated with the deposit address of this phonon. This key is needed to withdraw (or spend) the phonon.

#### `assetId`

Internal identifier for the type of asset. This can be a network token such as BTC or ETH or a token such as an ERC20 or ERC721. This value does not need to be validated by a counterparty.

#### `amount`

The amount to transact. For Bitcoin, this uint16 represents the total number of satoshis (the atomic unit). For Ethereum, this may not be enough to describe the atomic units and must be combined with `decimals`.

> A 2-byte value is used for two reasons. First, it allows storage of more phonons on each card. Second, it forces more "human" denominations, such as 1, 5, 10, etc. As with fiat bills, the presence of more standardized denominations is generally favorable for exchange, simplicity, and also privacy.


#### `decimals` (ETH ONLY)

If the coin has >8 decimals, it must utilize this field in the following way:

```
atomicUnits = amount * (10 ** decimals)
```

It is up to the depositor to encode these two parameters. So long as the recipient can verify the full amount on-chain, any equivalent multiplicative combination is allowed. For example, the following are equivalent to represent 6.12 ether (which has 18 decimals):

1. `amount = 612`, `decimals = 16`
2. `amount = 6,120,000`, `decimals = 12`

#### `extraData`

33 bytes of data used to describe the transaction or asset. 

* For Bitcoin, this is the transaction hash and a 1-byte `vout` (output) index of this UTXO within the transaction that created it.
* For Ethereum, this contains an identifier hash (left-padded with one zero) indicating what bespoke token this is. Generally this pertains to non-fungible tokens (NFTs) such as ERC721s.

> `extraData` may be left blank in some cases, but it is generally preferred so that recipients can more easily verify the authenticity of a phonon.

#### `networkId`

An index that maps to a network descriptor on the card. In the case of Ethereum, this is the settlement contract address. For Bitcoin, it is usually 0 or null.

It is important to note that the actual descriptor on the card is used for verification by the recipient of a Phonon Network transction. The sender and recipient must agree on the mappings between network ids and network descriptors in order to agree on the value of the phonon being transferred.

## Storing Network References

The card must keep a number of network "references", which are 32-byte descriptors indexed on a network ID:

```
private byte[] networks;
```

Here, each network descriptor is a 32-byte slice of `networks`, which is a 1D array of size `numNetworks * 32`. For example, if we want the identifier for network 3, we would slice `networks[96:128]`.

> For the near future, descriptors will likely be either 20 bytes (Ethereum contract addresses - left padded with zeros) or 0 bytes (Bitcoin). However, the slot is kept at 32 bytes for upgradability.

The card owner may, at any time, update his card's network list using the following API pseudocode:

* Get a 32-byte network descriptor for the given slot. If it is empty, this is 32 zero bytes:
```
byte[] getNetworkReference(short id) {
  short i = 32*id;
  return networks[i:32+i];
}
```

* Set a 32-byte slice of `networks` using the provided `reference` at the provided index (`32*id`):
```
void setNetworkReference(short id, byte[] reference) {
  short i = 32*id;
  networks[i:32+i] = reference;
}
```

## Referencing Asset IDs

The `assetId` parameter is for internal accounting of assets held by a card. Its value is entirely subjective and is only referenced by the user's interface to the card. If a user wishes for `assetId=0` to represent Ethereum and `assetId=1` to represent Bitcoin, that is equivalent to the reverse.

When combined with the network references specified above, a user can easily determine which network this phonon pertains to. A typical transfer might be prefaced by the sender providing the relevant network reference in addition to the static phonon (i.e. not containing the private key). However, the exact proof mechanism is left up to the implementor.

## Deposits

A phonon can be added to the card with the following pseudocode:

```
deposit(..params) {

  // Check that the deposit address corresponds to a nonce that is higher
  // than the current global nonce.
  validateDeposit(nonce);

  // Derive the private key associated with the deposit address
  // using the same nonce index
  byte[] priv = deriveIdPriv(nonce);
  
  // Instantiate the phonon and put it in the next available storage slot
  Phonon p = new Phonon(priv, assetId, amount, decimals, extraData, networkId);
  phonons[nextAvailableIndex] = p;
  
}
```

This will re-derive the recipient private key using the index. Recall that this was previously derived so that the user could generate a deposit *address*. We now use the corresponding *private key* as the identifier.

The included data is packed into a `Phonon` and is stored at the first unused index.

> Note that there is a maximum number of phonons which may be stored on a given card. If the card runs out of space, this API call will fail, but the user can store the phonon somewhere else and send it to the card at any time - of course this means the user must also persist the `nonce`, which is not part of the phonon deposit metadata!

### Nonce Tracking

Recall the presence of `globalNonce` to track the last deposit address index that was used. Any deposit **must** be based on a nonce **greater than** the global nonce at that time:

```
validateDeposit(long n) {
  if (n <= globalNonce) {
    throw new Error();
  } else {
    globalNonce = n;
  }
}
```

This mechanism prevents against replay attacks, whereby a user could deposit the same phonon multiple times - including after sending it. However, the user's software should be careful to issue deposits **in order** of the corresponding deposit address index (or else the `n <= globalNonce` condition will not be satisfied and the deposit will be void).

### Network Specificity and Verification

On Ethereum, deposits will correspond to calls to a smart contract, which is responsible for managing balances, deposits, and withdrawals. On Bitcoin and other UTXO-based networks, deposits are simply transfers to the deposit address (i.e. they look like any other transaction on the network).

You may have noticed that *no proofs are needed for deposits*. This is because on the Phonon Network, it is the responsibility of the **recipient** to validate the following when receiving a phonon:

1. The counterparty has a card that the recipient trusts (based on its certificate, which must be signed by the same card issuer).
2. The phonon has been deposited on the correct network and corresponds to the correct asset id.
3. The value of the phonon is correct.
4. The deposit has a sufficient amount of work (or, more generally, finality) behind it.

If these conditions are satisfied, the recipient can send an "ack" message to the sender (covered later) indicating that he is satisfied with the payment.

## Withdrawals

The card may withdraw a phonon at any time by passing the phonon index and calling the "withdraw" function:

```
byte[] withdraw(short phononIndex, byte[] data) {
  Phonon p = phonons[phononIndex];  
  // Make sure there is a valid phonon at the specified index
  if (p == null) {
    throw new Error();
  }
  // Sign whatever needs to be signed with the phonon's private key
  byte[] sig = p.sign(data);
  // Delete the phonon and return the signed data
  phonons[phononIndex] = null;
  return sig;
}
```

This looks up the phonon based on a storage index, which simply indicates where in the card's memory this data is being stored. If no phonon exists at that index, this call will fail.

`data` contains withdrawal parameters, which is the receiving address for Ethereum and is a script for Bitcoin. Other blockchain networks (or layer two protocols) may also potentially use Phonon by requesting different signature data for withdrawal.

### Ethereum-based Withdrawals

For Ethereum, the data to be signed depends on the smart contract. For the simplest case, a user would withdraw based on a receiving address. Once this address was signed by the phonon (which then self-destructed), it could be presented to a smart contract by another Ethereum account. This would trigger the contract to transfer the phonon coins based on the signer's address (i.e. the phonon's address) and then delete the record.

> Other, more complex designs could also be used if the withdrawal mechanism required it. Again, the phonon simply signs whatever data is presented.

#### Aside: Ethereum Signature Recovery Parameter

Ethereum smart contracts do not have the ability to *verify* a signature relative to a public key, but they are able to *recover* a signer based on a signature and message. They do this using a [recovery parameter](https://ethereum.stackexchange.com/questions/57478/generate-v-parameter-in-ethereum-transaction). This is a single bit (value 0 or 1), represented by `v`, which is usually thrown away when a signature is serialized (e.g. using DER format).

Smart cards return a 64-byte signature containing `r` and `s` params, which are 32 bytes each. It does **not** return a `v` value. Therefore, we need to recreate `v`. Although it isn't very well documented, there *is* a way to calculate `v`: see [here](https://ethereum.stackexchange.com/a/53182). 

However, because it is a single bit, we can avoid on-card complexity and just brute force the value on another layer using a mechanism such as the following:

```
let v = 27;
if (pubKey == eth.ecrecover(msg, 27, sig.r, sig.s).toString('hex')) {
  console.log('v is 27');
} else if (pubKey == eth.ecrecover(msg, 28, sig.r, sig.s).toString('hex')) {
  console.log('v is 28');
} else {
  console.log('Signature is invalid');
}
```

The above uses [`ethereumjs-util`](https://www.npmjs.com/package/ethereumjs-util) to perform `ecrecover` on a signature. Although it expects either 27 or 28 for `v`, you can see that its range is still binary. Don't worry too much about the actual values - just find which ones your library is expecting.

Once requesting a withdrawal signature, the interface must do a check such as the above to determine `v` before passing the withdrawal data to the on-chain smart contract.


### Bitcoin Withdrawals

Bitcoin withdrawals must unencumber the coins by signing the transaction input. The simplest design would likely rely on [segregated witness inputs](https://github.com/bitcoinbook/bitcoinbook/blob/develop/ch07.asciidoc#segregated-witness), which do not need to know state of other inputs in a transaction.

> **IMPORTANT NOTE:** *Unlike Ethereum, Bitcoin withdrawal data does **not** include the recipient! The recipient must be specified in the transaction output by the interface, making a secure interface like the [Lattice](https://gridplus.io/lattice) more important than it is for Ethereum.*

# Transacting on the Phonon Network

Phonons are sent across the Phonon Network using encrypted communication channels that connect any two cards. Because transfers are validated between only two participants, there is no concept of blocks or block times. Because phonon transfers are atomic, "consensus" is generally achieved out-of-band. 

## Connections

In order for cards on the Phonon Network to communicate, they must form encrypted communication channels using their identity keypairs. Shared secrets are derived via ECDH and are used for symmetric AES encryption.

### Using a Receipt Salt

All messages are encrypted end-to-end using a shared secret derived from the two cards' identity keys. However, transfers require a *second* layer of encryption using the recipient's `receiptSalt`, a temporary random number used to prevent replay attacks.

Generally, a card should come with a fixed number of receipt salts and may want to associate them with timestamps in case their corresponding transfers are never sent.

### Sending a Phonon

Once the normal communication channel (which uses a shared secret derived from the two identity keys) is established, the sender sends a message to the recipient indicating he is ready to send one or more phonons. The recipient's interface performs the necessary logic to ensure there is a new `receiptSalt` that the sender can use to encrypt the phonon packet.

> It is **very** important that the recipient generates a **new** `receiptSalt` and does not re-use an existing one. When a phonon packet is received that utilizes a `receiptSalt`, that `receiptSalt` is deleted!

The following mechanism generally outlines how the phonon salting and encrypting mechanism works:

0. Bob is going to transfer a phonon to Alice.
1. Alice calls `genReceiptSaltAtIndex()`, at an empty index to create a new salt.
2. Alice's card generates the following private key: `sha256(receiptSalt, idPrivate)` and then its public key, which is returned and sent to Bob (along with the salt index)
3. Bob's card generates a random key pair (`randPriv` and `randPub`) and combines `randPub` with Alice's salt-based public key from step 2 to generate a shared secret (`sec`) using ECDH.
4. Bob's card serializes and encrypts the phonon (`encPhonon`) using `sec` from step 3 and then deletes the phonon object from storage.
5. Bob sends the payload `[randomPub][encPhonon]` to Alice.
5. Alice's card re-creates `sec` with Bob's `randPub` and her salted private key, which must be re-derived. `sec` is used to AES-decrypt the phonon payload.
6. Alice's card deletes the relevant `receiptSalt` and stores the phonon.

### Viewing Static Phonon Data

Because the phonon is *deleted* when the transfer is initiated, the sender could find himself in a situation where the recipient doesn't agree to the value or network of the phonon and the sender would lose that phonon forever!

To avoid this situation, the card may also serialize the above data **without the private key** (and with the *public key* instead). Using this option, the user may send phonon data for verification by the counterparty without sending the phonon itself.

> Sending static phonon data does **not** require a salted shared secret - it is only encrypted once, using the normal secure channel built from identity keys.

### A Few Notes on Human Errors and Malicious Activity

Phonon deposits, transfers and withdrawals are all functions that lead to *immediate finality*. For example, transfers and withdrawals both **delete** the phonon from storage. If this is done by mistake, the phonon is gone forever. As such, there are several types of situations where a user could find himself in a bad situation.

In this section we cover a few pitfalls, which all point to the fact that it is wise to transact with *relatively small values* in Phonon.

#### Malicious Recipients

Unlike networks like Ethereum and Bitcoin, there is no way to *prove* that a card received a phonon (besides interfacing with it directly, of course). This means that a sender could find himself in a situation where he sends an encrypted phonon payload and the recipient claims to have never received it (or claims that the encryption key was incorrect and the phonon could not be deserialized).

#### Malicious Senders

Although Phonon prevents replay and double spend attacks, recipients must be diligent in being on the same page about network and token type. An unsuspecting recipient may, for instance, receive a phonon containing ETC (Ethereum Classic) tokens while expecting ETH (Ethereum) tokens. These would be indistinguishable, but for the network descriptor. 

Of course this is not unique to Phonon and is something cryptocurrency users (or, more precisely, their wallet software) are likely already used to.

#### Nonces and Salts

As mentioned previously, it is extremely important that the user's Phonon interface software manages nonces and salts properly. If a user wishes to deposit 50 phonons to 50 sequential addresses, these must all be deposited in the proper order. This is because of the `globalNonce` discussed earlier. If phonon #5 is deposited before phonon #4, the nonce will be incremented and phonon #4 will be impossible to deposit.

Similarly, if a user is expecting a phonon which will be encrypted based on a specific receipt salt, he must not re-use that salt. If Alice gives the public key corresponding to salt #1 to both Bob and Carol and each of them transfers a phonon out using it, only one of those phonons may be received. The other will be impossible to decrypt because the salt is deleted once it is used once.
