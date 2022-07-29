## How to run node on every clouds, every operating systems

**Problem:**
- Install Guide only for Ubuntu.
- No containerization.

**My Solution**
- The easiest way to run in every clouds is build your own container.
- There are a lot of node install guides in the internet, so i will not tell about it more here. Just logged my action. 
- Key point is the idea of containerization everythings here. Then your can ship and run everywhere.

## 000. Prepare

```
Install docker https://docs.docker.com/engine/install/
mkdir /root/near/
docker run -id --name stake-wars -v /root/near/:/root/.near/ --net host ubuntu:focal
```

## 001. Create your Shardnet wallet

```
docker exec -ti stake-wars bash
apt update && apt upgrade -y
apt install sudo
apt install curl
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -  
sudo apt install build-essential nodejs
PATH="$PATH"
node -v
npm -v
sudo npm install -g near-cli
export NEAR_ENV=shardnet
echo 'export NEAR_ENV=shardnet' >> ~/.bashrc
near proposals
near validators current
near validators next
```


## 002. Setup a validator and sync it to the actual state of the network.

```
lscpu | grep -P '(?=.*avx )(?=.*sse4.2 )(?=.*cx16 )(?=.*popcnt )' > /dev/null \
  && echo "Supported" \
  || echo "Not supported"

sudo apt install -y git binutils-dev libcurl4-openssl-dev zlib1g-dev libdw-dev libiberty-dev cmake gcc g++ python docker.io protobuf-compiler libssl-dev pkg-config clang llvm cargo

sudo apt install python3-pip

USER_BASE_BIN=$(python3 -m site --user-base)/bin
export PATH="$USER_BASE_BIN:$PATH"

sudo apt install clang build-essential make

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

source $HOME/.cargo/env

cd /
git clone https://github.com/near/nearcore
cd nearcore
git fetch

git checkout c1b047b8187accbf6bd16539feb7bb60185bdc38

cargo build -p neard --release --features shardnet
./target/release/neard --home ~/.near init --chain-id shardnet --download-genesis

rm ~/.near/config.json
wget -O ~/.near/config.json https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/shardnet/config.json

cd ~/.near
wget https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/shardnet/genesis.json

cd /nearcore
./target/release/neard --home ~/.near run

near login

pool_id='ducnc.factory.shardnet.near'
YOUR_WALLET='ducnc.shardnet.near'
near generate-key $pool_id
cp ~/.near-credentials/shardnet/${YOUR_WALLET}.json ~/.near/validator_key.json
sed -i 's/private_key/secret_key/g' ~/.near/validator_key.json


cd /nearcore
./target/release/neard --home ~/.near run
```

```
docker stop stake-wars
docker commit stake-wars stake-wars:latest
docker images | grep stake-wars:latest
docker rm stake-wars
docker run -d --name stake-wars --net host --entrypoint "/nearcore/target/release/neard" -v /root/near/:/root/.near/ --restart always stake-wars:latest run
docker logs -f -n 100 stake-wars
```

## 003. Deploy a new staking pool for your validator.

```
docker exec -ti stake-wars bash
near call ducnc.factory.shardnet.near unstake '{"amount": "1"}' --accountId ducnc.shardnet.near --gas=300000000000000
near call ducnc.factory.shardnet.near deposit_and_stake --amount 1 --accountId ducnc.shardnet.near --gas=300000000000000
near view ducnc.factory.shardnet.near get_account_staked_balance '{"account_id": "ducnc.shardnet.near"}'
near view ducnc.factory.shardnet.near get_account_unstaked_balance '{"account_id": "ducnc.shardnet.near"}'
near view ducnc.factory.shardnet.near get_account_total_balance '{"account_id": "ducnc.shardnet.near"}'
near call ducnc.factory.shardnet.near ping '{}' --accountId ducnc.shardnet.near --gas=300000000000000
```

## 004. Setup tools for monitoring node status.

```
sudo apt install curl jq
docker exec -ti stake-wars bash
curl -s http://127.0.0.1:3030/status | jq .version
near view ducnc.factory.shardnet.near get_accounts '{"from_index": 0, "limit": 10}' --accountId ducnc.shardnet.near
curl -s -d '{"jsonrpc": "2.0", "method": "validators", "id": "dontcare", "params": [null]}' -H 'Content-Type: application/json' 127.0.0.1:3030 | jq -c '.result.prev_epoch_kickout[] | select(.account_id | contains ("ducnc.factory.shardnet.near"))' | jq .reason
curl -s -d '{"jsonrpc": "2.0", "method": "validators", "id": "dontcare", "params": [null]}' -H 'Content-Type: application/json' 127.0.0.1:3030 | jq -c '.result.current_validators[] | select(.account_id | contains ("ducnc.factory.shardnet.near"))'
```
