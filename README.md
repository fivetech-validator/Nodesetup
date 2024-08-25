# Nodesetup
Install dependencies
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
Install Go
```
cd $HOME
sudo rm -rf /usr/local/go
wget https://go.dev/dl/go1.21.11.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.11.linux-amd64.tar.gz
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source ~/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
go version
which go
rm -rf go1.21.11.linux-amd64.tar.gz
```

Download and build binary
```
cd $HOME
rm -rf chain
git clone https://github.com/bandprotocol/chain
cd chain
git checkout v2.5.4
make install
bandd version
which bandd
```

Initialize the node
```
bandd config chain-id laozi-mainnet
```
Initialize the node
```
bandd init you-node-name --chain-id laozi-mainnet
```
Download genesis and addrbook
```
curl -Ls https://snapshots.tienthuattoan.com/mainnet/band/genesis.json > $HOME/.band/config/genesis.json
curl -Ls https://snapshots.tienthuattoan.com/mainnet/band/addrbook.json > $HOME/.band/config/addrbook.json
```
Add seed
```
sed -i -e "s|^seeds *=.*|seeds = \"cb8d586b2d48ec19d288aa44c3e50939ef68aa83@band-mainnet-rpc.tienthuattoan.com:32656\"|" $HOME/.band/config/config.toml
```
Set minimum gas price
```
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.0025uband\"|" $HOME/.band/config/app.toml
```
Update pruning and disable indexer to save disk space
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "10"|' \
  $HOME/.band/config/app.toml

sed -i 's/indexer = "kv"/indexer = "null"/g' $HOME/.band/config/config.toml
```
Create service
```
sudo tee /etc/systemd/system/bandd.service > /dev/null <<EOF
[Unit]
Description=Band node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which bandd) start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

Start the service and check the logs
```
sudo systemctl daemon-reload
sudo systemctl enable bandd
sudo systemctl restart bandd
sudo journalctl -u bandd -f --no-hostname -o cat
```
Key management
Add new key
```
bandd keys add wallet
```
Recover existing key
```
bandd keys add wallet --recover
```
List all keys
```
bandd keys list
```
Delete key
```
bandd keys delete your-wallet
```
Export key to a file
```
bandd keys export wallet
```
Import key from a file
```
bandd keys import wallet wallet.backup
```
Query your-wallet balance
```
bandd q bank balances $(bandd keys show your-wallet -a)
```

Validator management
Create new validator

```
bandd tx staking create-validator \
--amount 1000000uband \
--pubkey $(bandd tendermint show-validator) \
--moniker "FIVETECH" \
--identity "495CADCFB1CC4C00" \
--details "Blockchain | Node & Validator" \
--website "https://explorer.fivetech.pro" \
--chain-id laozi-mainnet \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.05 \
--min-self-delegation 1 \
--from your-wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.0025uband \
-y
```

Edit existing validator
```
bandd tx staking edit-validator \
--new-moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id laozi-mainnet \
--commission-rate 0.05 \
--from your-wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.0025uband \
-y
```
Unjail validator
```
bandd tx slashing unjail --from wallet --chain-id laozi-mainnet --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Jail reason
```
bandd query slashing signing-info $(bandd tendermint show-validator)
```
List all active validators
```
bandd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```
List all inactive validators
```
bandd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_UNBONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```
View validator details
```
bandd q staking validator $(bandd keys show your-wallet --bech val -a)
```
Token management
Withdraw rewards from all validators
```
bandd tx distribution withdraw-all-rewards --from your-wallet --chain-id bandd --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Withdraw commission and rewards from your validator
```
bandd tx distribution withdraw-rewards $(bandd keys show your-wallet --bech val -a) --commission --from your-wallet --chain-id bandd --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Delegate tokens to yourself
```
bandd tx staking delegate $(bandd keys show your-wallet --bech val -a) 1000000uom --from your-wallet --chain-id bandd --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Delegate tokens to validator
```
bandd tx staking delegate  1000000uom --from your-wallet --chain-id bandd --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Redelegate tokens to another validator
```
bandd tx staking redelegate $(bandd keys show your-wallet --bech val -a)  1000000uom --from your-wallet --chain-id bandd --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Unbond tokens from your validator
```
bandd tx staking unbond $(bandd keys show your-wallet --bech val -a) 1000000uom --from your-wallet --chain-id bandd --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Send tokens to the your-wallet
```
bandd tx bank send your-wallet  1000000uom --from your-wallet --chain-id bandd --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Governance
List all proposals
```
bandd query gov proposals
```
View proposal by id
```
bandd query gov proposal 1
```
Vote ‘Yes’
```
bandd tx gov vote 1 yes --from your-wallet --chain-id laozi-mainnet --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Vote ‘No’
```
bandd tx gov vote 1 no --from your-wallet --chain-id laozi-mainnet --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Vote ‘Abstain’
```
bandd tx gov vote 1 abstain --from your-wallet --chain-id laozi-mainnet --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Vote ‘NoWithVeto’
```
bandd tx gov vote 1 NoWithVeto --from your-wallet --chain-id laozi-mainnet --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uband -y
```
Utility
Update ports CUSTOM_PORT=110
```
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${CUSTOM_PORT}58\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${CUSTOM_PORT}57\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${CUSTOM_PORT}60\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${CUSTOM_PORT}56\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${CUSTOM_PORT}66\"%" $HOME/.band/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${CUSTOM_PORT}17\"%; s%^address = \":8080\"%address = \":${CUSTOM_PORT}80\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${CUSTOM_PORT}90\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${CUSTOM_PORT}91\"%" $HOME/.band/config/app.toml
```
Update Indexer
Disable indexer
```
sed -i -e 's|^indexer *=.*|indexer = "null"|' $HOME/.band/config/config.toml
```
Enable indexer
```
sed -i -e 's|^indexer *=.*|indexer = "kv"|' $HOME/.band/config/config.toml
```
Update pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.band/config/app.toml
```
Maintenance
Get validator info
```
bandd status 2>&1 | jq .ValidatorInfo
```
Get sync info
```
bandd status 2>&1 | jq .SyncInfo
```
Get node peer
```
echo $(bandd tendermint show-node-id)'@'$(curl -s ifconfig.me)':'$(cat $HOME/.band/config/config.toml | sed -n '/Address to listen for incoming connection/{n;p;}' | sed 's/.*://; s/".*//')
```
Check if validator key is correct
```
[ $(bandd q staking validator $(bandd keys show your-wallet --bech val -a) -oj | jq -r .consensus_pubkey.key) = $(bandd status | jq -r .ValidatorInfo.PubKey.value) ]] && echo -e "\n\e[1m\e[32mTrue\e[0m\n" || echo -e "\n\e[1m\e[31mFalse\e[0m\n"
```
Get live peers
```
curl -sS http://localhost:26657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```
Set minimum gas price
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0025uband\"/" $HOME/.band/config/app.toml
```
Enable prometheus
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.band/config/config.toml
```
Reset chain data
```
bandd tendermint unsafe-reset-all --home $HOME/.band --keep-addr-book
```
Remove node
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json
```
cd $HOME
sudo systemctl stop bandd
sudo systemctl disable bandd
sudo rm /etc/systemd/system/bandd.service
sudo systemctl daemon-reload
rm -f $(which bandd)
rm -rf $HOME/.band
```
Service Management
Reload service configuration
```
sudo systemctl daemon-reload
```
Enable service
```
sudo systemctl enable bandd
```
Disable service
```
sudo systemctl disable bandd
```
Start service
```
sudo systemctl start bandd
```
Stop service
```
sudo systemctl stop bandd
```
Restart service
```
sudo systemctl restart bandd
```
Check service status
```
sudo systemctl status bandd
```
Check service logs
```
sudo journalctl -u bandd -f --no-hostname -o cat
```


Band Node Snapshot ( EDIT HTTP)
```
sudo systemctl stop bandd

cp $HOME/.band/data/priv_validator_state.json $HOME/.band/priv_validator_state.json.backup
curl https://snapshots.polkachu.com/snapshots/band/band_32146450.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.band
mv $HOME/.band/priv_validator_state.json.backup $HOME/.band/data/priv_validator_state.json

sudo systemctl restart bandd && sudo journalctl -u bandd -f --no-hostname -o cat
Copy
cp $HOME/.band/data/priv_validator_state.json $HOME/.band/priv_validator_state.json.backup
bandd tendermint unsafe-reset-all --home $HOME/.band

SNAP_RPC="https://band-grpc.polkachu.com:22990"

LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height);
BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000));
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash) 

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH && sleep 2

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ;
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ;
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ;
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ;
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.band/config/config.toml

cp $HOME/.band/priv_validator_state.json.backup $HOME/.band/data/priv_validator_state.json

sudo systemctl restart bandd && sudo journalctl -u bandd -f
```
