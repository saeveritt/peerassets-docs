---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - python
  - shell

toc_footers:
  - <a href='#'>Donate to PeerAssets.org</a>

includes:
  - errors

search: true
---

# Introduction

Welcome to PeerAssets, a blockchain agnostic protocol which enables peers to issue, transact, burn, and vote with assets! You can use our API, papi.peercoin.net, or host your own! These allow you to access PeerAssets API endpoints, which contains information pertaining to PeerAssets Decks, Cards, and Votes. PeerAssets is inspired by the original idea of "Colored Coins" and uses OP_RETURN to write data on the blockchain, but offers some optimizations to reduce the amount of data written in OP_RETURN. PeerAssets enables easy querying of the blockchain for relevant transactions and offers some extra features like shareholder voting, dividends payouts, and more! To read more about PeerAssets and its functionality check out the [Whitepaper](https://peerassets.github.io/WhitePaper/)!


You can view code examples in the dark area to the right, and you can switch the programming language of the examples with the tabs in the top right.

# Protocol


## Protocol Buffer
PeerAssets utilizes a protocol buffer that is language-neutral, platform-neutral, and acts as an extensible mechanism for serializing structured data.
This project currently uses Google's [proto3](https://developers.google.com/protocol-buffers/docs/proto3) language. With proto3, versions of the PeerAssets protocol can be readily
created to work with Go, Ruby, Python, Objective-C, C++, and C# implementations.

The `.proto` file can be located [here](https://github.com/PeerAssets/peerassets-rfcs/blob/master/0001-peerassets-transaction-specification.proto).

```python
from pypeerassets.paproto_pb2 import DeckSpawn

Deck = DeckSpawn()
Deck.version = 1
Deck.name = 'My Deck'
Deck.number_of_decimals = 8
Deck.issue_mode = 0x04
Deck.asset_specific_data = 'Hello World'
Deck.SerializeToString()
```
**Example of a serialized Deck Object:**  
  (28 bytes)  
  `b'\x08\x01\x12\x07My Deck\x18\x08 \x04*\x0bHello World'`

This data is embedded into OP_RETURN and only takes up 28 bytes. 

**Example not using the protocol buffer**  
(101 bytes)  
`b'version: 1, name: "My Deck", number_of_decimals: 8, issue_mode: 4, asset_specific_data: "Hello World"'`

This optimization allows PeerAssets to be a compact protocol with many advantages over current colored coin implementations. Without using proto3 we would have to store approximately 100 bytes of data.


## Pay-to-TagHash
Querying for non-indexed data on the blockchain can be considered inefficient and should be avoided. Pay-to-TagHash, denoted P2TH, is the methodology in which all PeerAssets transactions are indexed. This allows thin clients to find PeerAssets related transactions rapidly using standard functions exposed by widely available blockchain explorers and rpc-nodes. You can read about the full mechanism by checking out the [Whitepaper](https://peerassets.github.io/P2TH/).

**Example:**  
Say an issuer with the address `n1ga7fPBerBZDK9NPru39Fanzj9cPhbumM` creates a DeckSpawn transaction and submits it to the network. This transaction will have a 256 bit transaction id. In this case the id for this issuer's transaction is `17d24b9bca5a090a24af138c2e085f80621396e8c7b6f820dee7140aee15cac1`. We can take advantage of this unique transaction id in a few ways. We want to look at this 256 bit value as a publicly known privatekey. Although this may seem counterintuative it allows us to do a few things.

> Generate WIF and Obtain P2TH Address

```python
import pypeerassets as pa
from binascii import unhexlify

privkey = unhexlify('7d24b9bca5a090a24af138c2e085f80621396e8c7b6f820dee7140aee15cac1')
KeyObject = pa.Kutil(privkey=privkey, network='tppc')
wif = KeyObject.wif
addr = KeyObject.address
```

For those using a rpc-node as the data source, a [WIF](https://en.bitcoin.it/wiki/Wallet_import_format) of this transaction id should be generated so that you may import it into your local node. Second, we will obtain the address, denoted the P2TH address, associated with this publicly known private key. _Note that this is not the same address as the issuer's address._  

**_Please refer to the code block titled "Generate WIF and Obtain the P2TH Address"._**  
`wif  = 'U5uhNJbLp22qRARNNJ954NTDzV3HUXMqQmKETtis95P9GtiqddKN'`  
`addr = 'PHUAHFBhFRypCbnTiXgGHiYyrftzkvDxVE'`

The reason for taking the DeckSpawn transaction id as a publicly known privatekey is to allow anyone to import the key into their node in order to track this deck.
The mechanism that makes this useful is the fact that every valid PeerAssets transaction for that deck pays a fee to the P2TH address, in this instance `PHUAHFBhFRypCbnTiXgGHiYyrftzkvDxVE`. Since this address is included in every valid PeerAssets transaction then importing the wif belonging to this address will scan the
blockchain and download all relevant transactions. Another reason for creating this deterministic address for tagging transactions is that any standard blockexplorer can be used to find relevant PeerAssets transactions. Simply list all transaction for the given P2TH address.  
  
**_P2TH Directory Address_**
- A network specific address that designates where all PeerAssets DeckSpawns will be indexed. In the case of Peercoin-Testnet the address is _miHhMLaMWubq4Wx6SdTEqZcUHEGp8RKMZt_.

## Transaction Structure    

### DeckSpawn
For a DeckSpawn transaction consider the following ,  

- Use of `vout[0]` is strictly for tagging the P2TH Directory Address  
- Use of `vout[1]` is strictly for OP_RETURN data  

  
**_Please refer to the code block titled "Example of what vout looks like for a DeckSpawn transaction"_**  
You will see two vouts, vout[0] and vout[1]. The important parts of vout[0] is the value, which is the P2TH fee, and the receiving address _miHhMLaMWubq4Wx6SdTEqZcUHEGp8RKMZt_, which in this case is the production P2TH directory address for Peercoin-Testnet. 

> Example what the vouts look like for a DeckSpawn transaction

```json
"vout" : [
        {
            "value" : 0.01000000,
            "n" : 0,
            "scriptPubKey" : {
                "asm" : "OP_DUP OP_HASH160 1e667ee94ea8e62c63fe59a0269bb3c091c86ca3 OP_EQUALVERIFY OP_CHECKSIG",
                "hex" : "76a9141e667ee94ea8e62c63fe59a0269bb3c091c86ca388ac",
                "reqSigs" : 1,
                "type" : "pubkeyhash",
                "addresses" : [
                    "miHhMLaMWubq4Wx6SdTEqZcUHEGp8RKMZt"
                ]
            }
        },
        {
            "value" : 0.00000000,
            "n" : 1,
            "scriptPubKey" : {
                "asm" : "OP_RETURN 0801120677696c6c794218022004",
                "hex" : "6a0e0801120677696c6c794218022004",
                "type" : "nulldata"
            }
        },
    ]
```
Located in vout[1] is a 0 value output that contains an OP_RETURN script. Inside the OP_RETURN field of the example we find the following hexadecimal data:     `0801120677696c6c794218022004`  
  
Let's take a look at the binary form of this data and parse the information with our protobuf objects.  

**_Please refer to the code block titled "Example of parsing the hexdata into a DeckSpawn object"._**  

To see the result,  

**_Please refer to the code block titled "Parsed from string using the DeckSpawn object in pypeerassets"._**


> Example of parsing the hexdata into a DeckSpawn object

```python
import pypeerassets as pa
from binascii import unhexlify

op_return = unhexlify('0801120677696c6c794218022004')
Deck = pa.paproto_pb2.DeckSpawn()
Deck.ParseFromString(op_return)
Deck
```

> Parsed from string using the DeckSpawn object in pypeerassets.

```
version: 1
name: "willyB"
number_of_decimals: 2
issue_mode: 4

```



### CardTransfer
For a CardTransfer transaction consider the following ,

- Use of `vout[0]` is strictly for tagging the P2TH Address  
- Use of `vout[1]` is strictly for OP_RETURN data  
- 0 Value use of a `vout[n] where n > 1` is to designate the receiver referenced in the amount fields of OP_RETURN data.  


> Example of a Card Transaction's vout:

```json
{
"vout" : [
        {
            "value" : 0.01000000,
            "n" : 0,
            "scriptPubKey" : {
                "asm" : "OP_DUP OP_HASH160 f2c67a919521a0031493bcd73687567ddee3f072 OP_EQUALVERIFY OP_CHECKSIG",
                "hex" : "76a914f2c67a919521a0031493bcd73687567ddee3f07288ac",
                "reqSigs" : 1,
                "type" : "pubkeyhash",
                "addresses" : [
                    "< P2TH Address >"
                ]
            }
        },
        {
            "value" : 0.00000000,
            "n" : 1,
            "scriptPubKey" : {
                "asm" : "OP_RETURN 080112178c89f806cba48c03c0921785a1a602a0f8eb02b2ceea011802",
                "hex" : "6a1d080112178c89f806cba48c03c0921785a1a602a0f8eb02b2ceea011802",
                "type" : "nulldata"
            }
        },
        {
            "value" : 0.00000000,
            "n" : 2,
            "scriptPubKey" : {
                "asm" : "OP_DUP OP_HASH160 33f9ce209f35cb842395b025b22fec0596933ac1 OP_EQUALVERIFY OP_CHECKSIG",
                "hex" : "76a91433f9ce209f35cb842395b025b22fec0596933ac188ac",
                "reqSigs" : 1,
                "type" : "pubkeyhash",
                "addresses" : [
                    "< Receiving Address 1 >"
                ]
            }
        },
        {
            "value" : 0.00000000,
            "n" : 3,
            "scriptPubKey" : {
                "asm" : "OP_DUP OP_HASH160 1325dfefd4ef7b38fdd3e0c1b6941502b7891889 OP_EQUALVERIFY OP_CHECKSIG",
                "hex" : "76a9141325dfefd4ef7b38fdd3e0c1b6941502b789188988ac",
                "reqSigs" : 1,
                "type" : "pubkeyhash",
                "addresses" : [
                    "< Receiving Address 2 ... >"
                ]
            }
        }
    ]
}
```

> Example of what's inside CardTransfer OP_RETURN

```python
import pypeerassets as pa
from binascii import unhexlify

op_return = unhexlify('080112178c89f806cba48c03c0921785a1a602a0f8eb02b2ceea011802')
Card = pa.paproto_pb2.CardTransfer()
Card.ParseFromString(op_return)
Card
```
**_Please refer to the code block titled "Example of a Card Transaction's vout"_**  
Lets' take a look at the OP_RETURN data in vout[1].  
`080112178c89f806cba48c03c0921785a1a602a0f8eb02b2ceea011802`  


> Parsed from string using the CardTransfer object in pypeerassets.

```
version: 1  
amount: 14550156  
amount: 6492747  
amount: 379200  
amount: 4821125  
amount: 5962784  
amount: 3843890  
number_of_decimals: 2
```

The first instance of `amount` in the parsed data will represent the amount being transfered from the sender to the receiver designated in the addresses field of vout[2].
The second instance of `amount` in the parsed data will represent the amount being transfered to the receiver defined in the addresses field of vout[3] and so
on. For this example this means that there should be a total of 8 vout's ( though the number has been reduced in the printed example to save space).  
  
- vout[0] : Pays Fee to P2TH Address
- vout[1] : 0 Value OP_RETURN Containing Serialized Card Data
- vout[n..] : 0 Value pubkeyhash sent to receiving address that corresponds to amount[n] located in Serialized Card Data





## Decks 
On the right you can see an example of what a PeerAssets Deck Object looks like. It contains the fields version, name, number_of_decimals, issue_mode, asset_specific_data, and fee.

> An example of a PeerAssets Deck Object,

```
{
    "version" : 1,
    "name" : "My Deck",
    "number_of_decimals": 8,
    "issue_mode": 0x04,
    "asset_specific_data": "",
    "fee": 0
  }
```
### Version
The `version` field represents the PeerAssets protocol version number used to define this deck.

### Name
The `name` field defines the name which will be assigned to the Deck.

### Number of Decimals
The `number_of_decimals` field defines the precision level of a Deck. It represents the number of decimals in which an 
asset can be divided into. For example, Bitcoin uses 8. 

### Issue Modes
- `NONE   = 0x00;`  
  - No issuance allowed  
- `CUSTOM = 0x01;`  
  - Not specified, custom client implementation needed  
- `ONCE   = 0x02;`  
  - Only one issuance transaction from asset owner allowed  
- `MULTI  = 0x04;`  
  - Multiple issuance transactions from asset owner allowed  
- `MONO   = 0x08;`  
  - All card transaction amounts are equal to 1  
- `UNFLUSHABLE  = 0x10;`  
  - No card transfer transactions allowed except for the card-issue transaction 0x20 used by SUBSCRIPTION (0x34 = 0x20 | 0x04 | 0x10)  
- `SUBSCRIPTION = 0x34;`  
  - An address is subscribed to a service for X hours since the first received cards. 
Where X is the asset balance of the address. This mode automatically enables MULTI & UNFLUSHABLE (0x34 = 0x20 | 0x04 | 0x10)  
- `SINGLET = 0x0a;`  
  - SINGLET is a combination of ONCE and MONO (0x02 | 0x08). Singlet deck, one MONO card issunce allowed. (ONCE | MONO)  

### Asset Specific Data

### Fee


## Cards

# Papi

<aside class="notice">
You must have the Docker service running and Docker-Compose installed.
</aside>

> Install Docker-CE

```shell
$ sudo apt-get update
$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu (lsb_release -cs) stable"
$ sudo apt-get update
$ sudo apt-get install docker-ce
```

> Install Docker-Compose

```shell
$ sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m)\
 -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

> Install Papi:

```shell
$ git clone https://github.com/PeerAssets/papi
$ cd papi
$ sudo service docker start
$ docker-compose up
```


### HTTP Request

* **API:**  
`GET http://papi.peercoin.net/api/v1/decks`  
`GET http://papi.peercoin.net/api/v1/decks/<deck_id>`  
`GET http://papi.peercoin.net/api/v1/decks/<deck_id>/balances`  

* **REST API:**  
`GET http://papi.peercoin.net/restless/v1/<__tablename__>`

### Table: "decks"  

Parameter | Description | Type
--------- | ------- | -----------
id | Deck Transaction ID | String
name | Deck Name | String
issuer | Deck Issuer's Address | String
issue_mode | Deck Issue Mode | Integer
decimals | Deck Precision | Integer
subscribed | Server Subscription Status | Boolean
asset_specific_data | Freeform Data | String

### Table: "cards"  

Parameter | Description | Type
--------- | ------- | -----------
txid | Card Transaction ID | String
blockhash | Blockhash of Card | String
blocknum | Blocknum of Card | Integer
blockseq | Transaction Sequence in Block | Integer
cardseq | Card Sequence in Transaction | Integer
sender | Card Sender's Address | String
receiver | Card Receivers Address | String
amount | Card Amount | BigInteger
ctype | Card Transaction Type | String
valid | Validity State | Boolean
asset_specific_data | Freeform Data | String

### Table: "balances"  

Parameter | Description | Type
--------- | ------- | -----------
id | Deck Transaction ID | String
name | Deck Name | String
issuer | Deck Issuer's Address | String
issue_mode | Deck Issue Mode | Integer
decimals | Deck Precision | Integer
subscribed | Server Subscription Status | Boolean
asset_specific_data | Freeform Data | String


### Query Parameters

## Get a Specific Deck


> The above command returns JSON structured like this:

```json
{
  "asset_specific_data": "", 
  "cards": [
    {
      "amount": 5, 
      "asset_specific_data": "", 
      "blockhash": "3a5e2fcb6db8ebd5286797dbcbf867de70f0f3d7808c7e41d8b3c994c9e183e7", 
      "blocknum": 340046, 
      "blockseq": 2, 
      "cardseq": 0, 
      "ctype": "CardIssue", 
      "deck_id": "d9ffff2db33c855b5092fa756069fce031b2f14d57f5d49541e7a51e025217e8", 
      "receiver": "mjdW8G6efg1a6PGF1PKcNrs5RTXQFc1aAs", 
      "sender": "n1ga7fPBerBZDK9NPru39Fanzj9cPhbumM", 
      "txid": "269ddc3330a883cb3711714b233d5431b1ab1bfb6b740ef07ce3bb053fcdeb7f", 
      "valid": true
    }
  ], 
  "decimals": 8, 
  "id": "d9ffff2db33c855b5092fa756069fce031b2f14d57f5d49541e7a51e025217e8", 
  "issue_mode": 4, 
  "issuer": "n1ga7fPBerBZDK9NPru39Fanzj9cPhbumM", 
  "name": "willyBtrippin", 
  "subscribed": true
}
```

This endpoint retrieves Cards for a specific Deck.

<aside class="warning">Testing the warning.</aside>

### HTTP Request

`GET http://papi.peercoin.net/api/v1/<deck_id>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the Deck to retrieve

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "deleted" : ":("
}
```

# Pypeerassets
A python implementaion of the PeerAssets Protocol. Pypeerassets aims to implement the PeerAssets protocol itself and to provide elementary interfaces to underlying blockchains. 
Requires Python3.5+.

## Installation
To install pypeerassets you can use `pip install pypeerassets` or directly clone the latest version from the official repository. Click on the shell tab
on the right and follow the listed commands.

```shell
git clone https://github.com/peerassets/pypeerassets
cd pypeerassets
python3 setup.py install --user
```

## Use
Pypeerassets can be used for direct interation with PeerAssets transactions in the blockchain.  For example, let us look how to list all of the current valid decks

> To use pypeerassets, try this code:

```python
import pypeerassets as pa
provider = pa.RpcNode(testnet=True)

pa.find_all_valid_decks( provider, version=1, production=True)
```