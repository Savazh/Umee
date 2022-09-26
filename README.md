LINKS:
Github
https://github.com/umee-network/testnets/tree/main/networks/canon-1
https://github.com/umee-network/umee#release-compatibility-matrix
https://mzonder.notion.site/UMEE-start-from-genesis-canon-1-8ac7abccfcd94d7d97431b0d1558bf8b

Explorers
https://explorer.network.umee.cc/canon-1

# 1. Install and sync umeed

### Install dependencies
# Update if needed
sudo apt update && sudo apt upgrade -y

# Insall packages
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu -y

# Install GO 1.18.5
cd $HOME
ver="1.18.5"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile

go version

# OUTPUT 
# go version go1.18.5 linux/amd64

BUILD UMEE
cd $HOME
cd umee || { git clone https://github.com/umee-network/umee.git && cd umee; }
git pull
git checkout v1.0.4
make install

umeed version
# v1.0.4

INIT UMEE
UMEE_CHAIN="canon-1"  # don't change

UMEE_NODENAME="MZONDER"
# UMEE_HOME="/mnt/disk1/.umee"
UMEE_HOME="$HOME/.umee"
MAIN_WALLET="MAIN_WALLET"
ORCH_WALLET="ORCH_WALLET"

echo "export UMEE_CHAIN=${UMEE_CHAIN}
export UMEE_NODENAME=${UMEE_NODENAME}
export UMEE_HOME=${UMEE_HOME}
export MAIN_WALLET=${MAIN_WALLET}
export ORCH_WALLET=${ORCH_WALLET}" >> $HOME/.bash_profile

source $HOME/.bash_profile

# init
umeed init ${UMEE_NODENAME} --chain-id $UMEE_CHAIN --home $UMEE_HOME

# genesis
wget -O $UMEE_HOME/config/genesis.json "https://raw.githubusercontent.com/umee-network/testnets/main/networks/canon-1/genesis.json"

# reset
umeed unsafe-reset-all --home $UMEE_HOME

CONFIG UMEE

# update peers
SEEDS=""
PEERS="5e01b69ead6e0781af0361d3ec4e436d96dba932@35.215.98.106:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $UMEE_HOME/config/config.toml

# min_gas
sed -i -e 's/^minimum-gas-prices *=.*/minimum-gas-prices = "0.001uumee"/' $UMEE_HOME/config/app.toml

CONFIG PRUNING AND SNAPSHOT (for validator nodes)

# config pruning
pruning_keep_recent="20000"
pruning_keep_every="0"
pruning_interval="19"

sed -i -e "s/^pruning *=.*/pruning = \"custom\"/;\
s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/;\
s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/;\
s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $UMEE_HOME/config/app.toml

# for pruning "100-0-10" you should disable snapshots to avoid conflict with 100 blocks pruning and 1500 blocks snapshot-interval
sed -i 's/snapshot-interval *=.*/snapshot-interval = 1000/' $UMEE_HOME/config/app.toml

CREATE AND RUN SERVICE

# create service
tee $HOME/umeed.service > /dev/null <<EOF
[Unit]
  Description=Umee
  After=network-online.target
[Service]
  User=$USER
  ExecStart=$(which umeed) start --home $UMEE_HOME
  Restart=on-failure
  RestartSec=10
  LimitNOFILE=65535
[Install]
  WantedBy=multi-user.target
EOF

sudo mv $HOME/umeed.service /etc/systemd/system/

sudo systemctl enable umeed
sudo systemctl daemon-reload
sudo systemctl start umeed && journalctl -u umeed -f -o cat

CREATE NEW KEY (or recover old one)

# create new key
umeed keys add $MAIN_WALLET --home $UMEE_HOME
umeed keys add $ORCH_WALLET --home $UMEE_HOME


# !!!! SAVE MNEMONICs FROM OUTPUT !!!!


# save addrs and valoper to vars (optional)
MAIN_ADDR=$(umeed keys show $MAIN_WALLET -a --home $UMEE_HOME)
echo $MAIN_ADDR
echo 'export MAIN_ADDR='${MAIN_ADDR} >> $HOME/.bash_profile

ORCH_ADDR=$(umeed keys show $ORCH_WALLET -a --home $UMEE_HOME)
echo $ORCH_ADDR
echo 'export ORCH_ADDR='${ORCH_ADDR} >> $HOME/.bash_profile

UMEE_VALOPER=$(umeed keys show $MAIN_WALLET --bech val -a --home $UMEE_HOME)

echo $UMEE_VALOPER
echo 'export UMEE_VALOPER='${UMEE_VALOPER} >> $HOME/.bash_profile

source $HOME/.bash_profile

**Syncing steps:**

Governance UPGRADE at block 339357

UPDATE UMEED

# stop
sudo systemctl stop umeed

# build
cd $HOME/umee
git pull
git checkout v1.1.2
make build

mv $HOME/umee/build/umeed $(which umeed)

# continue from 339357
sudo systemctl start umeed && journalctl -u umeed -f -o cat

# set halt-height = 416813

# continue from 398928
sudo systemctl start umeed && journalctl -u umeed -f -o cat

do 'umeed rollback' if CONSENSUS FAILURE (occured on block 400450)

## Halt-heighted UPGRADE at block 416813

UPDATE UMEED

# stop
sudo systemctl stop umeed

# build
cd $HOME/umee
git pull
git checkout v3.0.0-rc2
make build

mv $HOME/umee/build/umeed $(which umeed)

umeed version
# HEAD-7c5df71062a9b9f56342a07a15e087cee9a40128

set halt-height = 421777

# continue from 416814
sudo systemctl start umeed && journalctl -u umeed -f -o cat

## Halt-heighted UPGRADE at block `421777`

UPDATE UMEED

# stop
sudo systemctl stop umeed

# build
cd $HOME/umee
git pull
git checkout v3.0.0-rc3
make build

mv $HOME/umee/build/umeed $(which umeed)

umeed version
# HEAD-421da96607b66302c75feedc2e40092e70f2e10c

set halt-height = 0

# continue from 421777
sudo systemctl start umeed && journalctl -u umeed -f -o cat

WAIT FOR SYNC

# Before continue make sure you are fully synced with the network
umeed status 2>&1 | jq

2. CREATE VALIDATOR

# check balance
umeed q bank balances $MAIN_ADDR --home $UMEE_HOME

# check vars
echo $UMEE_NODENAME,$UMEE_CHAIN,$MAIN_WALLET,$UMEE_HOME | tr "," "\n" | nl 
# = 4 lines

# send tx
umeed tx staking create-validator \
  --amount 50000000uumee \
  --pubkey `umeed tendermint show-validator` \
  --moniker $UMEE_NODENAME \
  --commission-rate 0.1 \
  --commission-max-rate 0.5 \
  --commission-max-change-rate 0.1 \
  --min-self-delegation 1 \
  --from $MAIN_WALLET \
  --chain-id $UMEE_CHAIN \
  --gas 300000 \
  --fees 1000uumee \
  --home $UMEE_HOME

# check validator
umeed q staking validator $UMEE_VALOPER --home $UMEE_HOME



# Download and save priv_validator_key.json on your PC!



# output active set
umeed q staking validators --limit 1000 -oj --home $UMEE_HOME \
 | jq -r '.validators[] | select(.status=="BOND_STATUS_BONDED") | [(.tokens|tonumber / pow(10;6)), .description.moniker] | @csv' \
 | column -t -s"," | tr -d '"'| sort -k1 -n -r | nl

umeed q staking validators --limit 1000 -oj --home $UMEE_HOME \
 | jq -r '.validators[] | [(.tokens|tonumber / pow(10;6)), .description.moniker, .operator_address, .status, .jailed] | @csv' \
 | column -t -s"," | tr -d '"'| sort -k1 -n -r | nl


# edit if needed 931A47B46A0D32AA 
umeed tx staking edit-validator --identity "YOU_KEY_BASE" --moniker $UMEE_NODENAME --from $MAIN_WALLET --home $UMEE_HOME

3. INSTALL PEGGO

cd $HOME
cd peggo || { git clone https://github.com/umee-network/peggo.git && cd peggo; }
git pull
git checkout v0.4.1
make install

peggo version

# version: v0.4.1
# commit: b22361c4821dceb9d1a67d42be4b344fefef5eca
# sdk: v0.46.0-umee.0.20220812010629-4d5bb2e3f73c
# go: go1.18.3 linux/amd64

# fund orchestrator wallet
umeed tx bank send $MAIN_ADDR $ORCH_ADDR 1000000uumee --chain-id $UMEE_CHAIN --fees 200uumee --home $UMEE_HOME


# FUND ETH addr


# Set up your variables
#                 START_HEIGHT="14211966"          
#                 --bridge-start-height=\"$START_HEIGHT\" \

BRIDGE_ADDR="0xa22344153e826580e679712648e487Dc6032fB4d"                          # don't change!  Umee Testnet canon-1 Gravity Address 0xa22344153e826580e679712648e487Dc6032fB4d

PEGGO_ETH_PK="78bcffxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" # Private key of PEGGO_ETH_ADDR (0x7d318xxxxxxxxxxxxxxxxxxxxxxxxxxxx)
ETH_RPC="https://goerli.infura.io/v3/xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
ALCHEMY_ENDPOINT="wss://eth-goerli.g.alchemy.com/v2/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

ORCHESTRATOR_WALLET_NAME="ORCH_WALLET"
KEYRING_PASSWORD="xxxxxxxxxxxxxx"

KEYRING="os"

# create service
tee $HOME/peggod.service > /dev/null <<EOF
Description=Peggo Service
After=network.target
[Service]
User=$USER
Type=simple
ExecStart=$(which peggo) orchestrator $BRIDGE_ADDR \
  --eth-rpc="$ETH_RPC" \
  --relay-batches=true \
  --valset-relay-mode=minimum \
  --cosmos-gas-prices="0.0025uumee" \
  --cosmos-chain-id=$UMEE_CHAIN \
  --cosmos-keyring-dir="$UMEE_HOME" \
  --cosmos-keyring="$KEYRING" \
  --cosmos-from="$ORCHESTRATOR_WALLET_NAME" \
  --cosmos-from-passphrase="$KEYRING_PASSWORD" \
  --eth-alchemy-ws="$ALCHEMY_ENDPOINT" \
  --log-level debug \
  --log-format text
Environment="PEGGO_ETH_PK=$PEGGO_ETH_PK"
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

sudo mv $HOME/peggod.service /etc/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable peggod
sudo systemctl start peggod && journalctl -u peggod -f -o cat

4. REGISTER ORCHESTRATOR KEYS

# your ETH addr
PEGGO_ETH_ADDR="0x7d318xxxxxxxxxxxxxxxxxxxxxxxxxx"
echo 'export PEGGO_ETH_ADDR='${PEGGO_ETH_ADDR} >> $HOME/.bash_profile

# check vars
echo $UMEE_VALOPER,$ORCH_ADDR,$PEGGO_ETH_ADDR,$MAIN_WALLET,$UMEE_CHAIN,$UMEE_HOME | tr "," "\n" | nl
# 6 lines

umeed tx gravity set-orchestrator-address $UMEE_VALOPER $ORCH_ADDR $PEGGO_ETH_ADDR --from $MAIN_WALLET --chain-id $UMEE_CHAIN  --gas 600000 --fees=6000000uumee --home $UMEE_HOME

# check peggo
sudo systemctl restart peggod && journalctl -u peggod -f -o cat

5. INSTALL PRICE-FEEDER

### build price-feeder

cd $HOME/umee/price-feeder
git pull
git checkout price-feeder/v1.0.0
make install

price-feeder version

# version: HEAD-ae66523e0521fe2e2f37175973d09033097a5a91
# commit: ae66523e0521fe2e2f37175973d09033097a5a91
# sdk: v0.46.1-umee
# go: go1.18.3 linux/amd64

CONFIG PRICE-FEEDER

# remove old config
rm -rf $HOME/price-feeder_config
mkdir -p $HOME/price-feeder_config

# set vars
UMEE_CHAIN="canon-1"
UMEE_VALOPER="umeevaloper1....."
MAIN_ADDR="umee1...validator_addr"

KEYRING="os"
KEYRING_PASSWORD="xxxxxxxxxx"

RPC_PORT="26657"
GRPC_PORT="9090"

# check vars
echo $UMEE_CHAIN, $UMEE_VALOPER, $MAIN_ADDR, $KEYRING, $KEYRING_PASSWORD, $RPC_PORT, $GRPC_PORT | tr "," "\n" | nl 
# output 7 lines

# create config
tee $HOME/price-feeder_config/price-feeder.toml > /dev/null <<EOF
gas_adjustment = 1

[server]
listen_addr = "0.0.0.0:7171"
read_timeout = "20s"
verbose_cors = true
write_timeout = "20s"

[[deviation_thresholds]]
base = "USDT"
threshold = "1.5"

[[deviation_thresholds]]
base = "UMEE"
threshold = "1.5"

[[deviation_thresholds]]
base = "ATOM"
threshold = "1.5"

[[currency_pairs]]
base = "UMEE"
providers = [
  "okx",
  "gate"
]
quote = "USDT"

[[currency_pairs]]
base = "UMEE"
providers = [
  "ftx"
]
quote = "USD"

[[currency_pairs]]
base = "USDT"
providers = [
  "kraken",
  "ftx",
  "coinbase"
]
quote = "USD"

[[currency_pairs]]
base = "ATOM"
providers = [
  "ftx",
  "okx",
]
quote = "USDT"

[[currency_pairs]]
base = "ATOM"
providers = [
  "kraken",
]
quote = "USD"

[account]
address = "${MAIN_ADDR}"
chain_id = "${UMEE_CHAIN}"
validator = "${UMEE_VALOPER}"

[keyring]
backend = "${KEYRING}"
dir = "$HOME/.umee"
pass = "${KEYRING_PASSWORD}"

[rpc]
grpc_endpoint = "localhost:${GRPC_PORT}"
rpc_timeout = "100ms"
tmrpc_endpoint = "http://localhost:${RPC_PORT}"

[telemetry]
enable-hostname = true
enable-hostname-label = true
enable-service-label = true
enabled = false
global_labels = [["chain-id", "${UMEE_CHAIN}"]]
service-name = "pfd"
prometheus-retention-time = 100

[[provider_endpoints]]
name = "binance"
rest = "https://api1.binance.com"
websocket = "stream.binance.com:9443"
EOF

# create service
echo "[Unit]
Description=Price feeder service
After=network.target
[Service]
User=$USER
Environment=PRICE_FEEDER_PASS=${KEYRING_PASSWORD}
Type=simple
ExecStart=$(which price-feeder) $HOME/price-feeder_config/price-feeder.toml --log-level debug
RestartSec=10
Restart=on-failure
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target" > $HOME/pfd.service

sudo mv $HOME/pfd.service /etc/systemd/system/

START PRICE-FEEDER


sudo systemctl daemon-reload
sudo systemctl enable pfd
sudo systemctl start pfd && journalctl -u pfd -f -o cat



https://mzonder.notion.site/UMEE-start-from-genesis-canon-1-8ac7abccfcd94d7d97431b0d1558bf8b
