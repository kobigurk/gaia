[![CircleCI](https://circleci.com/gh/cosmos/gaia/tree/master.svg?style=svg)](https://circleci.com/gh/cosmos/gaia/tree/master)

# Cosmos-Hub Staking Module

This project is a demonstration of the Cosmos-Hub with basic staking
functionality staking module designed to get validators acquainted
with staking concepts and procedures.

Currently, the validator set is updated every block. The validator set is
determined as the validators with the top 100 bonded atoms. Currently, all
bonding and unbonding is instantaneous (no queue). Absent features include,
delegation, validator rewards, unbonding wait period.

## Installation
```
go get github.com/cosmos/gaia 
cd $GOPATH/src/github.com/cosmos/gaia
make all
```

See the [cosmos-sdk documentation](https://cosmos-sdk.readthedocs.io) for more.

### Gaia-1 Test-Net Example

Gaia-1 is a community test-net which can be used to test setting up a cosmos validator node. 
Supplimentary to this tutorial you can also follow [this video](https://www.youtube.com/watch?v=B-shjoqvnnY).

To work on the Gaia-1 you will need some tokens to get started. 
To do this first generate a new key: 

```
MYNAME=<your name>
gaia client keys new $MYNAME
gaia client keys list
MYADDR=<your newly generated address>
```

Then enter your key into [this utility](http://www.cosmosvalidators.com/)
with your key address and it will send you some `fermion` testnet tokens :).
Fermions are the native staking token on Gaia-1 - see below for how to check your balance.


Now, to sync with the testnet, we need the genesis file and seeds.
The easiest way to get them is to Fetch and navigate to the tendermint testnet repo:
```
git clone https://github.com/tendermint/testnets $HOME/testnets
GAIANET=$HOME/testnets/gaia-1/gaia
cd $GAIANET
```

Now we can start a new node in the background - but note that it may take a decent 
chunk of time to sync with the existing testnet... go brew a pot of coffee! 

```
gaia node start --home=$GAIANET  &> gaia.log &
```

Of course, you can follow the logs to see progress with `tail -f gaia.log`.
Once blocks slow down to about one block per second, you're all caught up.

The `gaia start` command will automaticaly generate a validator private key found in
`$GAIANET/priv_validator.json`. Let's get the pubkey data for our validator
node. The pubkey is located under `"pub_key"{"data":` within the json file

```
cat $GAIANET/priv_validator.json 
PUBKEY=<your newly generated address>  
```

If you have a json parser like `jq`, you can get just the pubkey like so:

```
PUBKEY=$(cat $GAIANET/priv_validator.json | jq -r .pub_key.data)
```

Next let's initialize the gaia client to start interacting with the testnet:

```
gaia client init --chain-id=gaia --node=tcp://localhost:46657
```

And check our balance:

```
gaia client query account $MYADDR
```

We are now ready to bond some tokens:

```
gaia client tx bond --amount=5fermion --name=$MYNAME --pubkey=$PUBKEY
```

Bonding tokens means that your balance is tied up as _stake_. Don't worry,
you'll be able to get it back later. As soon as some tokens have been bonded
the validator node which we started earlier will have power in the network and
begin to participate in consensus!

We can now check the validator set and see that we are a part of the club!
```
gaia client query validators
```

Finally lets unbond to get back our tokens

```
gaia client tx unbond --amount=5fermion --name=$MYNAME
```

Remember to unbond before stopping your node!

### Local-Test Example

Here is a quick example to get you off your feet: 

First, generate a new key with a name, and save the address:

```
MYNAME=<your name>
gaia client keys new $MYNAME
gaia client keys list
MYADDR=<your newly generated address>
```
Now initialize a gaia chain:

```
gaia init $MYADDR --home=$HOME/.gaia1 --chain-id=test 
```

This will create all the files necessary to run a single node chain in `$HOME/.gaia1`:
a `priv_validator.json` file with the validators private key, and a `genesis.json` file 
with the list of validators and accounts. In this case, we have one random validator,
and `$MYADDR` is an independent account that has a bunch of coins.

We can add a second node on our local machine by initiating a node in a new directory,
and copying in the genesis:


```
gaia node init $MYADDR --home=$HOME/.gaia2 --chain-id=test
cp $HOME/.gaia1/genesis.json $HOME/.gaia2/genesis.json
```

We need to also modify `$HOME/.gaia2/config.toml` to set new seeds and ports. It should look like:

```
proxy_app = "tcp://127.0.0.1:46668"
moniker = "anonymous"
fast_sync = true
db_backend = "leveldb"
log_level = "state:info,*:error"

[rpc]
laddr = "tcp://0.0.0.0:46667"

[p2p]
laddr = "tcp://0.0.0.0:46666"
seeds = "0.0.0.0:46656"
```

Great, now that we've initialized the chains, we can start both nodes in the background:

```
gaia node start --home=$HOME/.gaia1  &> gaia1.log &
NODE1_PID=$!
gaia node start --home=$HOME/.gaia2  &> gaia2.log &
NODE2_PID=$!
```

Note we save the `PID` so we can later kill the processes.

Of course, you can peak at your logs with `tail gaia1.log`, or follow them 
for a bit with `tail -f gaia1.log`.

Now we can initialize a client for the first node, and look up our account:

```
gaiacli init --chain-id=test --node=tcp://localhost:46657
gaiacli query account $MYADDR
```

Nice. We can also lookup the validator set:

```
gaiacli query validators
```

Notice it's empty! This is because the initial validators are special - 
the app doesn't know about them, so they can't be removed. To see what
tendermint itself thinks the validator set is, use:

```
curl localhost:46657/validators
```

Ok, let's add the second node as a validator. First, we need the pubkey data:

```
cat $HOME/.gaia2/priv_validator.json 
```

If you have a json parser like `jq`, you can get just the pubkey:

```
cat $HOME/.gaia2/priv_validator.json | jq .pub_key.data
```

Now we can bond some coins to that pubkey:

```
gaia client tx bond --amount=10fermion --name=$MYNAME --pubkey=<validator pubkey>
```

We should see our account balance decrement, and the pubkey get added to the app's list of bonds:

```
gaia client query account $MYADDR
gaia client query validators
``` 

To confirm for certain the new validator is active, check tendermint:

```
curl localhost:46657/validators
```

If you now kill your second node, blocks will stop streaming in, because there aren't enough validators online.
Turn her back on and they will start streaming again.

Finally, to relinquish all your power, unbond some coins. You should see your
VotingPower reduce and your account balance increase.

```
gaia client tx unbond --amount=10fermion --name=$MYNAME
gaia client query validators
gaia client query account $MYADDR
``` 

Once you unbond enough, you will no longer be needed to make new blocks.
