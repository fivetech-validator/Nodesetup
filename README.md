# Nodesetup
NODE SEDA MAINNET

explorer: https://explorer.nodestake.org/seda/staking

twitter: https://x.com/sedaprotocol

discord: https://discord.com/invite/seda

Setup validator name
Replace YOUR_MONIKER_GOES_HERE with your validator name
```
MONIKER=FIVETECH
```
Install dependencies
UPDATE SYSTEM AND INSTALL BUILD TOOLS
```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```

INSTALL GO
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.22.3.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

Download and build binaries
```
# Clone project repository
cd $HOME
rm -rf seda-chain
git clone https://github.com/sedaprotocol/seda-chain.git
cd seda-chain
git checkout v0.1.1

# Build binaries
make build

# Prepare binaries for Cosmovisor
mkdir -p $HOME/.sedad/cosmovisor/genesis/bin
mv build/sedad $HOME/.sedad/cosmovisor/genesis/bin/
rm -rf build

# Create application symlinks
sudo ln -s $HOME/.sedad/cosmovisor/genesis $HOME/.sedad/cosmovisor/current -f
sudo ln -s $HOME/.sedad/cosmovisor/current/bin/sedad /usr/local/bin/sedad -f

Install Cosmovisor and create a service
# Download and install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

# Create service
```
sudo tee /etc/systemd/system/seda.service > /dev/null << EOF
[Unit]
Description=seda node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.sedad"
Environment="DAEMON_NAME=sedad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.sedad/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable seda.service
```

Initialize the node
```
# Set node configuration
sedad config chain-id seda-1
sedad config keyring-backend file
sedad config node tcp://localhost:17357

# Initialize the node
sedad init $MONIKER --chain-id seda-1

# Download genesis and addrbook
curl -Ls https://snapshots.kjnodes.com/seda/genesis.json > $HOME/.sedad/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/seda/addrbook.json > $HOME/.sedad/config/addrbook.json

# Add seeds
sed -i -e "s|^seeds *=.*|seeds = \"400f3d9e30b69e78a7fb891f60d76fa3c73f0ecc@seda.rpc.kjnodes.com:17359\"|" $HOME/.sedad/config/config.toml

# Set minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"10000000000aseda\"|" $HOME/.sedad/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.sedad/config/app.toml

# Set custom ports
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:17358\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:17357\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:17360\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:17356\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":17366\"%" $HOME/.sedad/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:17317\"%; s%^address = \":8080\"%address = \":17380\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:17390\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:17391\"%; s%:8545%:17345%; s%:8546%:17346%; s%:6065%:17365%" $HOME/.sedad/config/app.toml
```

Download latest chain snapshot 5H
```
curl -L https://snapshots.kjnodes.com/seda/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.sedad
[[ -f $HOME/.sedad/data/upgrade-info.json ]] && cp $HOME/.sedad/data/upgrade-info.json $HOME/.sedad/cosmovisor/genesis/upgrade-info.json
```


Start service and check the logs
```
sudo systemctl start seda.service && sudo journalctl -u seda.service -f --no-hostname -o cat
```



Snapshot

Stop the service and reset the data
```
sudo systemctl stop seda.service
cp $HOME/.sedad/data/priv_validator_state.json $HOME/.sedad/priv_validator_state.json.backup
rm -rf $HOME/.sedad/data
```

Download latest snapshot
```
curl -L https://snapshots.kjnodes.com/seda/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.sedad
mv $HOME/.sedad/priv_validator_state.json.backup $HOME/.sedad/data/priv_validator_state.json
```

Restart the service and check the log
```
sudo systemctl start seda.service && sudo journalctl -u seda.service -f --no-hostname -o cat
```


	State sync
State Sync cho phép một nút mới tham gia mạng bằng cách tìm nạp ảnh chụp nhanh trạng thái ứng dụng ở mức cao gần đây,
 thay vì tìm nạp và phát lại tất cả các khối lịch sử. 
Vì trạng thái ứng dụng thường nhỏ hơn nhiều so với các khối và việc khôi phục nó nhanh hơn nhiều so với việc phát lại 
các khối nên điều này có thể giảm thời gian đồng bộ hóa với mạng từ vài ngày xuống còn vài phút.

Stop the service and reset the data
```
sudo systemctl stop seda.service
cp $HOME/.sedad/data/priv_validator_state.json $HOME/.sedad/priv_validator_state.json.backup
sedad tendermint unsafe-reset-all --keep-addr-book --home $HOME/.sedad
```

Get and configure the state sync information
```
STATE_SYNC_RPC=https://seda.rpc.kjnodes.com:443
STATE_SYNC_PEER=d9bfa29e0cf9c4ce0cc9c26d98e5d97228f93b0b@seda.rpc.kjnodes.com:17356
LATEST_HEIGHT=$(curl -s $STATE_SYNC_RPC/block | jq -r .result.block.header.height)
SYNC_BLOCK_HEIGHT=$(($LATEST_HEIGHT - 1000))
SYNC_BLOCK_HASH=$(curl -s "$STATE_SYNC_RPC/block?height=$SYNC_BLOCK_HEIGHT" | jq -r .result.block_id.hash)

sed -i \
  -e "s|^enable *=.*|enable = true|" \
  -e "s|^rpc_servers *=.*|rpc_servers = \"$STATE_SYNC_RPC,$STATE_SYNC_RPC\"|" \
  -e "s|^trust_height *=.*|trust_height = $SYNC_BLOCK_HEIGHT|" \
  -e "s|^trust_hash *=.*|trust_hash = \"$SYNC_BLOCK_HASH\"|" \
  -e "s|^persistent_peers *=.*|persistent_peers = \"$STATE_SYNC_PEER\"|" \
  $HOME/.sedad/config/config.toml

mv $HOME/.sedad/priv_validator_state.json.backup $HOME/.sedad/data/priv_validator_state.json
```

(Optional) Download latest wasm
If the chain does not support copy of the wasm folder, you can download it manually.
```
curl -L https://snapshots.kjnodes.com/seda/wasm_latest.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.sedad
```
Restart the service and check the log
```
sudo systemctl start seda.service && sudo journalctl -u seda.service -f --no-hostname -o cat
```

 Key management
ADD NEW KEY
```
sedad keys add wallet
```
RECOVER EXISTING KEY
```
sedad keys add wallet --recover
```

LIST ALL KEYS
```
sedad keys list
```

DELETE KEY
```
sedad keys delete wallet
```

EXPORT KEY TO A FILE
```
sedad keys export wallet
```

IMPORT KEY FROM A FILE
```
sedad keys import wallet wallet.backup
```

QUERY WALLET BALANCE
```
sedad q bank balances $(sedad keys show wallet -a)
```

👷 Validator management
Please make sure you have adjusted moniker, identity, details and website to match your values.

Create Validator
Obtain your validator public key by running the following command:

```
sedad comet show-validator
```
The output will be similar to this (with a different key):

{"@type":"/cosmos.crypto.ed25519.PubKey","key":"lR1d7YBVK5jYijOfWVKRFoWCsS4dg3kagT7LB9GnG8I="}
```
Then, create a file named validator.json with the following content:

```
nano $HOME/validator.json
```
Change your info, from "pubkey" to "details"  and Save them
{    
    "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"i5rgI5rkbgFKty+RjuK5UP7bLAbDuykP9cXTTAPMeAc="},
    "amount": "1000000aseda",
    "moniker": "FIVETECH",
    "identity": "495CADCFB1CC4C00",
    "website": "https://linktr.ee/fivetech_validator",
    "security": "https://t.me/fivetech_validator",
    "details": "Crypto VN No1",
    "commission-rate": "0.01",
    "commission-max-rate": "0.2",
    "commission-max-change-rate": "0.01",
    "min-self-delegation": "1"
}
```

Finally, we're ready to submit the transaction to create the validator:
```
sedad tx staking create-validator $HOME/validator.json \
--from=wallet \
--chain-id=seda-1 \
--fees=2000000000000000aseda 
```

UNJAIL VALIDATOR
```
sedad tx slashing unjail --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

JAIL REASON
```
sedad query slashing signing-info $(sedad tendermint show-validator)
```

LIST ALL ACTIVE VALIDATORS
```
sedad q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

LIST ALL INACTIVE VALIDATORS
```
sedad q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_UNBONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

VIEW VALIDATOR DETAILS
```
sedad q staking validator $(sedad keys show wallet --bech val -a)
```

💲 Token management
WITHDRAW REWARDS FROM ALL VALIDATORS
```
sedad tx distribution withdraw-all-rewards --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

WITHDRAW COMMISSION AND REWARDS FROM YOUR VALIDATOR
```
sedad tx distribution withdraw-rewards $(sedad keys show wallet --bech val -a) --commission --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

DELEGATE TOKENS TO YOURSELF
```
sedad tx staking delegate $(sedad keys show wallet --bech val -a) 1000000aseda --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

DELEGATE TOKENS TO VALIDATOR
```
sedad tx staking delegate <TO_VALOPER_ADDRESS> 1000000aseda --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

REDELEGATE TOKENS TO ANOTHER VALIDATOR
```
sedad tx staking redelegate $(sedad keys show wallet --bech val -a) <TO_VALOPER_ADDRESS> 1000000aseda --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

UNBOND TOKENS FROM YOUR VALIDATOR
```
sedad tx staking unbond $(sedad keys show wallet --bech val -a) 1000000aseda --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

SEND TOKENS TO THE WALLET
```
sedad tx bank send wallet <TO_WALLET_ADDRESS> 1000000aseda --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```
🗳 Governance
LIST ALL PROPOSALS
```
sedad query gov proposals
```
VIEW PROPOSAL BY ID
```
sedad query gov proposal 1
```

VOTE ‘YES’
```
sedad tx gov vote 1 yes --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

VOTE ‘NO’
```
sedad tx gov vote 1 no --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

VOTE ‘ABSTAIN’
```
sedad tx gov vote 1 abstain --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```

VOTE ‘NOWITHVETO’
```
sedad tx gov vote 1 NoWithVeto --from wallet --chain-id seda-1 --gas-adjustment 1.4 --gas auto --gas-prices 10000000000aseda -y
```
⚡️ Utility
UPDATE PORTS
```
CUSTOM_PORT=110
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${CUSTOM_PORT}58\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${CUSTOM_PORT}57\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${CUSTOM_PORT}60\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${CUSTOM_PORT}56\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${CUSTOM_PORT}66\"%" $HOME/.sedad/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${CUSTOM_PORT}17\"%; s%^address = \":8080\"%address = \":${CUSTOM_PORT}80\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${CUSTOM_PORT}90\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${CUSTOM_PORT}91\"%" $HOME/.sedad/config/app.toml
```
UPDATE INDEXER
Disable indexer

```
sed -i -e 's|^indexer *=.*|indexer = "null"|' $HOME/.sedad/config/config.toml
```

Enable indexer
```
sed -i -e 's|^indexer *=.*|indexer = "kv"|' $HOME/.sedad/config/config.toml
```

UPDATE PRUNING
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.sedad/config/app.toml
```

🚨 Maintenance
GET VALIDATOR INFO
```
sedad status 2>&1 | jq .ValidatorInfo
```

GET SYNC INFO
```
sedad status 2>&1 | jq .SyncInfo
```

GET NODE PEER
```
echo $(sedad tendermint show-node-id)'@'$(curl -s ifconfig.me)':'$(cat $HOME/.sedad/config/config.toml | sed -n '/Address to listen for incoming connection/{n;p;}' | sed 's/.*://; s/".*//')
```

CHECK IF VALIDATOR KEY IS CORRECT
```
[[ $(sedad q staking validator $(sedad keys show wallet --bech val -a) -oj | jq -r .consensus_pubkey.key) = $(sedad status | jq -r .ValidatorInfo.PubKey.value) ]] && echo -e "\n\e[1m\e[32mTrue\e[0m\n" || echo -e "\n\e[1m\e[31mFalse\e[0m\n"
```

GET LIVE PEERS
```
curl -sS http://localhost:17357/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

SET MINIMUM GAS PRICE
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"10000000000aseda\"/" $HOME/.sedad/config/app.toml
```

ENABLE PROMETHEUS
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.sedad/config/config.toml
```

RESET CHAIN DATA
```
sedad tendermint unsafe-reset-all --keep-addr-book --home $HOME/.sedad --keep-addr-book
```

REMOVE NODE
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json!
```
cd $HOME
sudo systemctl stop seda.service
sudo systemctl disable seda.service
sudo rm /etc/systemd/system/seda.service
sudo systemctl daemon-reload
rm -f $(which sedad)
rm -rf $HOME/.sedad
rm -rf $HOME/seda-chain
```

⚙️ Service Management
RELOAD SERVICE CONFIGURATION
```
sudo systemctl daemon-reload
```

ENABLE SERVICE
```
sudo systemctl enable seda.service
```

DISABLE SERVICE
```
sudo systemctl disable seda.service
```

START SERVICE
```
sudo systemctl start seda.service
```

STOP SERVICE
```
sudo systemctl stop seda.service
```

RESTART SERVICE
```
sudo systemctl restart seda.service
```

CHECK SERVICE STATUS
```
sudo systemctl status seda.service
```

CHECK SERVICE LOGS
```
sudo journalctl -u seda.service -f --no-hostname -o cat
```
CÁC DỮ LIỆU NHƯ SNAPSHORT PRC VVV CHÚNG TÔI LẤY NGUỒN CỦA HTTPS://KJNODES.COM CÁC BẠN CÓ THỂ LẤY NGUỒN NHIỀU BÊN KHÁC THAY THẾ VÀO 
