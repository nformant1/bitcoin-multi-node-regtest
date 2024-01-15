# Multinode / Multiwallet Dogecoin regtest network

This repository allows you run a full dogecoin network in an isolated environment. It uses dogecoin's regtest capability to setup an isolated dogecoin network, and then uses docker to setup a network with 3 nodes.

This is useful because normally in regtest mode you would generate all coins in the same wallet as where you'd send the coins. With this setup, you can use one node to generate the coins and then send it to one of the other nodes, which can then again send it to another node to simulate more real-life dogecoin usage.

## Usage

Simple run

`docker-compose up`

to start all the containers. This will start the bitcoin nodes, and expose RPC on all of them. The nodes will run on the following ports:

| Node | P2P port * | RPC port * | RPC Username | RPC Password |
| --- | --- | --- | --- | ---|
| miner1 | 18500 | 18400 | bitcoin | bitcoin |
| node1 | 18501 | 18401 | bitcoin | bitcoin |
| node2 | 18502 | 18402 | bitcoin | bitcoin |

\* Port as exposed on the host running docker.

## Samples

Note these samples use `curl` to exercise the API, but this would usually be `dogecoin-cli`. We're using `curl` so we don't have a dependency on dogecoin in the host.

### Initial block count

Checks that the initial block count is 0.

```sh
nformant@dogehouse:~/dogecoin-docker$ ls -ll
total 24
-rwxr-xr-x 1 nformant nformant   473 Jan 15 21:16 Dockerfile
-rwxr-xr-x 1 nformant nformant 11663 Jan 15 21:16 README.md
-rwxr-xr-x 1 nformant nformant   530 Jan 15 21:16 docker-compose.yml
-rwxr-xr-x 1 nformant nformant    86 Jan 15 21:16 dogecoin.conf
nformant@dogehouse:~/dogecoin-docker$ docker-compose up -d
[+] Running 4/4
 ⠿ Network dogecoin-docker_default    Created                                                    0.8s
 ⠿ Container dogecoin-docker-node1-1  Started                                                    3.3s
 ⠿ Container dogecoin-docker-node2-1  Started                                                    1.9s
 ⠿ Container dogecoin-docker-miner-1  Started                                                    3.5s
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getblockcount"}' -u bitcoin:bitcoin localhost:18400
{"result":0,"error":null,"id":"1"}
```

### Check connected nodes

Check the "miner" node is connected to other nodes.

```sh
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getpeerinfo","params":[]}' -u bitcoin:bitcoin -s localhost:18400 | jq '.result[] | {addr, inbound} '
{
  "addr": "172.21.0.3:58306",
  "inbound": true
}
{
  "addr": "172.21.0.2:51056",
  "inbound": true
}
{
  "addr": "node1:18444",
  "inbound": false
}
{
  "addr": "node2:18444",
  "inbound": false
}
```

### Mine some blocks and see other nodes are updating their block count

```sh
# mine the blocks
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"generate","params":[101]}' -u bitcoin:bitcoin -s localhost:18400
{"result":["c3adb1c18e7c217b5bd89ced6c5fd011add7cbf9070c776bdc98fd8ee62c1bee",...
<snip> 
...,"9dccba3649d3c840755806d0eae7b09cf11c3e82f51cb62fca85a5a7f839014e"],"error":null,"id":"1"}

# check on node1
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getblockcount","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":101,"error":null,"id":"1"}

# check on node2
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getblockcount","params":[]}' -u bitcoin:bitcoin -s localhost:18402
{"result":101,"error":null,"id":"1"}
```

### Send dogecoin from miner to another node

Now we're going to generate an address in another node. Note that we use port **18401** (node1) instead of 18400:
```sh
# Mine blocks first
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"generate","params":[101]}' -u bitcoin:bitcoin -s localhost:18400
{"result":["57344d56c4c60085ab5a7b29f9dab83ad5988d92fb5462d477dc4daebb3ef088",...
<snip>
...,"84a761d20fcca7b13469aef02ff9b315c27a313339a845902bdb960a2fc59ead"],"error":null,"id":"1"}

# Check balance
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18400
{"result":71000000.00000000,"error":null,"id":"1"}

# Generate address on node1
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getnewaddress","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":"mkfiuy2HZKmWEpY3zp18M3d7LMisY9vTod","error":null,"id":"1"}

# Send from miner to node1
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"sendtoaddress","params":["mkfiuy2HZKmWEpY3zp18M3d7LMisY9vTod", "4.20"]}' -u bitcoin:bitcoin -s localhost:18400
{"result":"3c86e1da58641c00946163f11e11037cecd67734de96e979827b148cce2f701a","error":null,"id":"1"}

# Now, since the block was not yet mined, we usually don't see the balance yet, unless  we specify 0 confirmations.
# First with the default (1) confirmation:
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":0.00000000,"error":null,"id":"1"}

# No coins, so let's try 0 confirmations (the first parameter "" means default account)
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":["", 0]}' -u bitcoin:bitcoin -s localhost:18401
{"result":4.20000000,"error":null,"id":"1"}

# This also means that node1 has it in the mempool, which shows there is exactly one transaction in it
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getmempoolinfo","params":[]}' -u bitcoin:bitcoin -s localhost:18401 | jq .
{
  "result": {
    "size": 1,
    "bytes": 191,
    "usage": 1088,
    "maxmempool": 300000000,
    "mempoolminfee": 0
  },
  "error": null,
  "id": "1"
}

# Finally, let's mine the block and see that getbalance will show the balance by default.
# miner
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"generate","params":[1]}' -u bitcoin:bitcoin -s localhost:18400
{"result":["e3dbf7db5ebaf89c977a6c85eb6c2b45d6ec36cd6c903d98a31b6eabfd6970de"],"error":null,"id":"1"}

# node1:
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":4.20000000,"error":null,"id":"1"}

# miner
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18400
{"result":71499995.79808000,"error":null,"id":"1"}

# Some extras

# List wallet affecting transactions:
# miner
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"listtransactions","params":["", 150]}' -u bitcoin:bitcoin -s localhost:18400 | jq '.result [] | {amount, confirmations, txid, category}'
{
  "amount": 500000,
  "confirmations": 149,
  "txid": "c94690c1a3440f7c4e3746df7c9264d1ea3aad121ac2e9bd5b1523f60024ae77",
  "category": "generate"
}
<snip>
{
  "amount": -4.2,
  "confirmations": 1,
  "txid": "3c86e1da58641c00946163f11e11037cecd67734de96e979827b148cce2f701a",
  "category": "send"
}
{
  "amount": 250000.00192,
  "confirmations": 1,
  "txid": "c7fc41180285505661fe99b3c90b9e4b4de9a024b3bbac0827094f1f3e79a530",
  "category": "immature"
}

# node1
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"listtransactions","params":["", 150]}' -u bitcoin:bitcoin -s localhost:18401 | jq '.result [] | {amount, confirmations, txid, category}'
{
  "amount": 4.2,
  "confirmations": 1,
  "txid": "3c86e1da58641c00946163f11e11037cecd67734de96e979827b148cce2f701a",
  "category": "receive"
}

# node2
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"listtransactions","params":["", 150]}' -u bitcoin:bitcoin -s localhost:18402
{"result":[],"error":null,"id":"1"}

# Try getting the transaction
# node1:
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"gettransaction","params":["008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0"]}' -u bitcoin:bitcoin -s localhost:18401 | jq .
{
  "result": {
    "amount": 3.14,
    "confirmations": 1,
    "blockhash": "00f3b3923a14a69a78a419660e20d63a4f91370d6099c3bbbdc5daad953e2985",
    "blockindex": 1,
    "blocktime": 1523421980,
    "txid": "008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0",
    "walletconflicts": [],
    "time": 1523421666,
    "timereceived": 1523421666,
    "bip125-replaceable": "no",
    "details": [
      {
        "account": "",
        "address": "2N444M93zqwyhhFSDJxgzPL5h9XMdwLUibz",
        "category": "receive",
        "amount": 3.14,
        "label": "",
        "vout": 0
      }
    ],
    "hex": "0200000001487d3c16738785ec7c3e7f9e703c715c68c0562f2433b5eac61edc05c8b1d57b0000000048473044022065b2d5c649bb6882f73654398ebaa9bf29281088493ad56f1ee0400bfcbeb8b30220402fea8148de73c14ba332693f7d5c9e2f2bf4332299a2df87115fdabaf1607b01feffffff028042b7120000000017a914768cc2b85515737b398e2613fd7fe9c3d44f73f187d0a04e170100000017a9146a82eef49043551afd83cb286d801c0725ba24318765000000"
  },
  "error": null,
  "id": "1"
}

# node2, here it fails because that transaction is not in the wallet.
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"gettransaction","params":["3c86e1da58641c00946163f11e11037cecd67734de96e979827b148cce2f701a"]}' -u bitcoin:bitcoin -s localhost:18402 | jq .
{
  "result": null,
  "error": {
    "code": -5,
    "message": "Invalid or non-wallet transaction id"
  },
  "id": "1"
}

# however, using `getrawtransaction` and `decoderawtransactoin` on node2 does actually return it
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"getrawtransaction","params":["3c86e1da58641c00946163f11e11037cecd67734de96e979827b148cce2f701a"]}' -u bitcoin:bitcoin -s localhost:18402 | jq .
{
  "result": "0100000001761849deafdbeb9b10a31824bdd3a7c8c224b8c046a39813dd5fc05906e050f100000000484730440220222f938272fbdb66d14be6ab140f54076ecf7ac2c0994698a3d4ec039264440402203cf2abde529aece2f372f849699c1123000b9c4098ea1e401f979bc7d95e28ee01feffffff020081316f792d00001976a9148ac086badf41e47a852193c75e6861b4843df87988ac00b10819000000001976a914388133ecd6d87b108455ee107c0f07b6695360fd88acca000000",
  "error": null,
  "id": "1"
}
nformant@dogehouse:~/dogecoin-docker$ curl -d '{"jsonrpc":"2.0","id":"1","method":"decoderawtransaction","params":["0100000001761849deafdbeb9b10a31824bdd3a7c8c224b8c046a39813dd5fc05906e050f100000000484730440220222f938272fbdb66d14be6ab140f54076ecf7ac2c0994698a3d4ec039264440402203cf2abde529aece2f372f849699c1123000b9c4098ea1e401f979bc7d95e28ee01feffffff020081316f792d00001976a9148ac086badf41e47a852193c75e6861b4843df87988ac00b10819000000001976a914388133ecd6d87b108455ee107c0f07b6695360fd88acca000000"]}' -u bitcoin:bitcoin -s localhost:18402 | jq .
{
  "result": {
    "txid": "3c86e1da58641c00946163f11e11037cecd67734de96e979827b148cce2f701a",
    "hash": "3c86e1da58641c00946163f11e11037cecd67734de96e979827b148cce2f701a",
    "size": 191,
    "vsize": 191,
    "version": 1,
    "locktime": 202,
    "vin": [
      {
        "txid": "f150e00659c05fdd1398a346c0b824c2c8a7d3bd2418a3109bebdbafde491876",
        "vout": 0,
        "scriptSig": {
          "asm": "30440220222f938272fbdb66d14be6ab140f54076ecf7ac2c0994698a3d4ec039264440402203cf2abde529aece2f372f849699c1123000b9c4098ea1e401f979bc7d95e28ee[ALL]",
          "hex": "4730440220222f938272fbdb66d14be6ab140f54076ecf7ac2c0994698a3d4ec039264440402203cf2abde529aece2f372f849699c1123000b9c4098ea1e401f979bc7d95e28ee01"
        },
        "sequence": 4294967294
      }
    ],
    "vout": [
      {
        "value": 499995.79808,
        "n": 0,
        "scriptPubKey": {
          "asm": "OP_DUP OP_HASH160 8ac086badf41e47a852193c75e6861b4843df879 OP_EQUALVERIFY OP_CHECKSIG",
          "hex": "76a9148ac086badf41e47a852193c75e6861b4843df87988ac",
          "reqSigs": 1,
          "type": "pubkeyhash",
          "addresses": [
            "mtAcBVEqYHpmTzTykm3iJrEvaohtAXaV4D"
          ]
        }
      },
      {
        "value": 4.2,
        "n": 1,
        "scriptPubKey": {
          "asm": "OP_DUP OP_HASH160 388133ecd6d87b108455ee107c0f07b6695360fd OP_EQUALVERIFY OP_CHECKSIG",
          "hex": "76a914388133ecd6d87b108455ee107c0f07b6695360fd88ac",
          "reqSigs": 1,
          "type": "pubkeyhash",
          "addresses": [
            "mkfiuy2HZKmWEpY3zp18M3d7LMisY9vTod"
          ]
        }
      }
    ]
  },
  "error": null,
  "id": "1"
}

```


## Requirements

To run this you need both `docker` and `docker-compose`. It was tested on a clean Ubuntu 20.04 with the docker-ce package from Docker. 
