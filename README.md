# empower-ibc-transfer
```

```

## download binary (for ubuntu 22)

```
cd $HOME
mkdir -p $HOME/.hermes/bin

version="v1.5.0"
wget "https://github.com/informalsystems/ibc-rs/releases/download/${version}/hermes-${version}-x86_64-unknown-linux-gnu.tar.gz"

tar -C $HOME/.hermes/bin/ -vxzf hermes-${version}-x86_64-unknown-linux-gnu.tar.gz

rm hermes-$version-x86_64-unknown-linux-gnu.tar.gz

```

```
echo "export PATH=$PATH:$HOME/.hermes/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile

hermes version
```

```
tee $HOME/hermesd.service > /dev/null <<EOF
[Unit]
Description=HERMES
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which hermes) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

sudo mv $HOME/hermesd.service /etc/systemd/system/


```
## start service

```
sudo systemctl daemon-reload
sudo systemctl enable hermesd
```

## create hermes config // change moniker name

```
[global]
log_level = 'debug'
[mode]
[mode.clients]
enabled = true
refresh = true
misbehaviour = true
[mode.connections]
enabled = false
[mode.channels]
enabled = false
[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
tx_confirmation = false
auto_register_counterparty_payee = false
[rest]
enabled = false
host = '127.0.0.1'
port = 3000
[telemetry]
enabled = false
host = '127.0.0.1'
port = 3001


[[chains]]
id = 'osmo-test-5'
ccv_consumer_chain = false

rpc_addr = 'https://rpc.osmotest5.osmosis.zone:443'
grpc_addr = 'https://grpc.osmotest5.osmosis.zone:443'
websocket_addr = 'wss://rpc.osmotest5.osmosis.zone/websocket'
rpc_timeout = '10s'
trusted_node = false
batch_delay = '500ms'   # hermes default 500ms

account_prefix = 'osmo'
key_name = 'OSMO_TEST_REL_WALLET'
store_prefix = 'ibc'

default_gas = 800000
max_gas = 1600000
gas_price = { price = 0.025, denom = 'uosmo' }
gas_multiplier = 1.2

max_msg_num = 30
max_tx_size = 180000   # hermes default 2097152

clock_drift = '10s'
max_block_time = '30s'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }

memo_prefix = 'Geralt irlandali_turist#7300'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-155'], # circulus-1 channel-0
]


[[chains]]
id = 'circulus-1'
ccv_consumer_chain = false

rpc_addr = 'http://127.0.0.1:26657'
grpc_addr = 'http://127.0.0.1:9090'
websocket_addr = 'ws://127.0.0.1:26657/websocket'
rpc_timeout = '10s'
trusted_node = false
batch_delay = '500ms'

account_prefix = 'empower'
key_name = 'EMPOWER_TEST_REL_WALLET'
store_prefix = 'ibc'

default_gas = 800000
max_gas = 1600000
gas_price = { price = 0.01, denom = 'umpwr' }
gas_multiplier = 1.2

max_msg_num = 30
max_tx_size = 180000

clock_drift = '10s'
max_block_time = '30s'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }

memo_prefix = 'Geralt irlandali_turist#7300'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-0'], # osmosis channel-155
]


```
## test config
```
hermes config validate
```

## add relayer wallet
```
read mnemonic && echo "$mnemonic" > $HOME/.hermes/EMPOWER_TEST_REL_WALLET.txt
read mnemonic && echo "$mnemonic" > $HOME/.hermes/OSMO_TEST_REL_WALLET.txt
```

```
hermes keys add --key-name EMPOWER_TEST_REL_WALLET --chain circulus-1 --mnemonic-file $HOME/.hermes/EMPOWER_TEST_REL_WALLET.txt
hermes keys add --key-name OSMO_TEST_REL_WALLET   --chain osmo-test-5      --mnemonic-file $HOME/.hermes/OSMO_TEST_REL_WALLET.txt
```
```
sudo systemctl start hermesd && journalctl -u hermesd -f -o cat
```

## build osmosis binary (go 1.19 is required)
```
cd $HOME
git clone https://github.com/osmosis-labs/osmosis
cd osmosis
git checkout v15.1.2
make install
```

## import osmosis wallet

```
osmosisd keys add OSMO_TEST_REL_WALLET --recover
```

## send ft token

circulus-1 ---- osmo-test-5
```
hermes tx ft-transfer   --number-msgs 10   --key-name EMPOWER_TEST_REL_WALLET   --receiver osmo1s492paw57du0kpudjr7pfwt350cprma7rq3vt6   --denom umpwr   --timeout-seconds 30   --dst-chain osmo-test-5   --src-chain circulus-1   --src-port transfer   --src-channel channel-0   --amount 200
  
```
osmo-test-5 ---- circulus-1

```
hermes tx ft-transfer   --number-msgs 10   --key-name OSMO_TEST_REL_WALLET   --receiver empower1s492paw57du0kpudjr7pfwt350cprma7hhxh8k   --denom uosmo   --timeout-seconds 30   --dst-chain circulus-1   --src-chain osmo-test-5   --src-port transfer   --src-channel channel-155   --amount 2000

```

ibc transfer 
```
empowerd tx ibc-transfer transfer transfer channel-0 osmowallet 111umpwr --from=empowerwallet --fees 200umpwr 
osmosisd tx ibc-transfer transfer transfer channel-155 empowerwallet 5555uosmo --from=osmowallet --fees 5000uosmo --chain-id osmo-test-5 --keyring-backend test --node https://rpc.osmotest5.osmosis.zone:443
```

## update clients example
```
hermes update client --host-chain circulus-1 --client 07-tendermint-1
hermes update client --host-chain osmo-test-5 --client 07-tendermint-146
```
