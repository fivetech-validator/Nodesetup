# AURA MAINNET
Manual installation

```
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

installation Go

```
ver="1.22.2"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

Build 03.06.24

```
cd $HOME && mkdir -p go/bin/
git clone https://github.com/aura-nw/aura
cd aura
git checkout v0.8.3
make install
```

Initiation
```
aurad init fivetech --chain-id=aura_6322-2
aurad config chain-id aura_6322-2
```

Download Genesis
```
wget https://images.aura.network/aura_6322-2-genesis.tar.gz
tar -xzvf aura_6322-2-genesis.tar.gz
mv aura_6322-2-genesis.json $HOME/.aura/config/genesis.json
```

Set up the minimum gas price and Peers/Seeds/Filter peers/MaxPeers
```
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.001uaura\"/;" ~/.aura/config/app.toml
external_address=$(wget -qO- eth0.me)
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.aura/config/config.toml
peers=""
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.aura/config/config.toml
seeds=""
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.aura/config/config.toml
sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 50/g' $HOME/.aura/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 50/g' $HOME/.aura/config/config.toml
```

Pruning (optional)
```
pruning="custom" &&
pruning_keep_recent="1000" &&
pruning_keep_every="0" &&
pruning_interval="10" &&
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.aura/config/app.toml &&
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.aura/config/app.toml &&
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.aura/config/app.toml &&
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.aura/config/app.toml
```

Indexer (optional)
```
indexer="null" &&
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.aura/config/config.toml
```

Download addrbook

```
wget -O $HOME/.aura/config/addrbook.json "https://raw.githubusercontent.com/111STAVR111/props/main/Aura/addrbook.json"
```

Create a service file
```
sudo tee /etc/systemd/system/aurad.service > /dev/null <<EOF
[Unit]
Description=aurad
After=network-online.target

[Service]
User=$USER
ExecStart=$(which aurad) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

SnapShot Mainnet updated every 5 hours

```
cd $HOME
apt install lz4
sudo systemctl stop aurad
cp $HOME/.aura/data/priv_validator_state.json $HOME/.aura/priv_validator_state.json.backup
rm -rf $HOME/.aura/data
curl -o - -L https://aura.snapshot.stavr.tech/aura-snap.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.aura --strip-components 2
curl -o - -L https://aura.wasm.stavr.tech/wasm-aura.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.aura --strip-components 2
mv $HOME/.aura/priv_validator_state.json.backup $HOME/.aura/data/priv_validator_state.json
wget -O $HOME/.aura/config/addrbook.json "https://raw.githubusercontent.com/111STAVR111/props/main/Aura/addrbook.json"
sudo systemctl restart aurad && journalctl -u aurad -f -o cat
```

Start
```
sudo systemctl daemon-reload
sudo systemctl enable aurad
sudo systemctl restart aurad && sudo journalctl -fu aurad -o cat
```

Create/recover wallet
```
aurad keys add wallet
```
           OR
```
aurad keys add wallet --recover
```

Create validator
```
aurad tx staking create-validator \
--amount 1000000ueaura \
--from wallet \
--commission-max-change-rate "0.1" \
--commission-max-rate "0.2" \
--commission-rate "0.05" \
--min-self-delegation "1" \
--details="" \
--identity="" \
--pubkey  $(aurad tendermint show-validator) \
--moniker FIVETECH \
--fees 555uaura \
--chain-id aura_6322-2 -y
```

Delete node
```
sudo systemctl stop aurad
sudo systemctl disable aurad
rm /etc/systemd/system/aurad.service
sudo systemctl daemon-reload
cd $HOME
rm -rf .aura
rm -rf aura
rm -rf $(which aurad)
```

Upgrade
```
cd $HOME/aura && git pull
git checkout v0.8.3
make install
aurad version --long | grep -e commit -e version
#version: v0.8.3
#commit: f904836400b0fb2bedccf6283f5c0b720d8aa932
sudo systemctl restart aurad && journalctl -fu aurad -o cat
#consensus
curl -s http://localhost:26657/consensus_state  | jq '.result.round_state.height_vote_set[0].prevotes_bit_array'
```

Service
Info

```
aurad status 2>&1 | jq .NodeInfo
aurad status 2>&1 | jq .SyncInfo
aurad status 2>&1 | jq .ValidatorInfo
```

Check node logs

```
sudo journalctl -fu aurad -o cat
```

Check service status
```
sudo systemctl status aurad 
```

Restart service
```
sudo systemctl restart aurad 
```

Stop service
```
sudo systemctl stop aurad 
```


Start service
```
sudo systemctl start aurad 
```

reload/disable/enable
```
sudo systemctl daemon-reload
sudo systemctl disable aurad 
sudo systemctl enable aurad 
```

Your Peer
```
echo $(aurad tendermint show-node-id)'@'$(wget -qO- eth0.me)':'$(cat $HOME/.aura/config/config.toml | sed -n '/Address to listen for incoming connection/{n;p;}' | sed 's/.*://; s/".*//')


Check all keys
```
aurad keys list
```

Check Balance
```
aurad q bank balances $(aurad keys show wallet -a)
```

Delete Key
```
aurad keys delete wallet
```

Export Key
```
aurad keys export wallet
```

Import Key
```
aurad keys import wallet wallet.backup
```


Edit Validator
```
aurad tx staking edit-validator \
--new-moniker "Your_Moniker" \
--identity "Keybase_ID" \
--details "Your_Description" \
--website "Your_Website" \
--security-contact "Your_Email" \
--chain-id aura_6322-2 \
--commission-rate 0.05 \
--from wallet \
--gas 350000 -y
```

Your Valoper-Address
```
aurad keys show Wallet_Name --bech val
```

Your Valcons-Address
```
aurad tendermint show-address
```

Unjail
```
aurad tx slashing unjail --from wallet --chain-id aura_6322-2 --gas 350000 --fees "97500"uaura -y
```
