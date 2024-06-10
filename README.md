# Nodesetup
# Nibiru
This manual offers a detailed, step-by-step approach for establishing and operating a full validator node on the Junction.

Website: https://nibiru.fi/


Twitter: https://twitter.com/NibiruChain

Discord: https://discord.com/invite/nibirufi


#### Services

https://explorer.fivetech.pro/Nibiru-mainnet

#Setup validator name
```
MONIKER=FIVETECH
```

#Install dependencies UPDATE SYSTEM AND INSTALL BUILD TOOLS
```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```

#INSTALL GO
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.21.10.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

#Download and build binaries
```
cd $HOME
rm -rf nibiru
git clone https://github.com/NibiruChain/nibiru.git
cd nibiru
git checkout v1.4.0

make build

mkdir -p $HOME/.nibid/cosmovisor/genesis/bin
mv build/nibid $HOME/.nibid/cosmovisor/genesis/bin/
rm -rf build

sudo ln -s $HOME/.nibid/cosmovisor/genesis $HOME/.nibid/cosmovisor/current -f
sudo ln -s $HOME/.nibid/cosmovisor/current/bin/nibid /usr/local/bin/nibid -f

#Install Cosmovisor and create a service

go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0


sudo tee /etc/systemd/system/nibiru.service > /dev/null << EOF
[Unit]
Description=nibiru node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.nibid"
Environment="DAEMON_NAME=nibid"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.nibid/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable nibiru.service
```

#Initialize the node
```
nibid config chain-id cataclysm-1
nibid config keyring-backend file
nibid config node tcp://localhost:13957


nibid init $MONIKER --chain-id cataclysm-1

curl -Ls https://snapshots.kjnodes.com/nibiru/genesis.json > $HOME/.nibid/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/nibiru/addrbook.json > $HOME/.nibid/config/addrbook.json


sed -i -e "s|^seeds *=.*|seeds = \"400f3d9e30b69e78a7fb891f60d76fa3c73f0ecc@nibiru.rpc.kjnodes.com:13959\"|" $HOME/.nibid/config/config.toml


sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025unibi\"|" $HOME/.nibid/config/app.toml

sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.nibid/config/app.toml


sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:13958\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:13957\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:13960\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:13956\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":13966\"%" $HOME/.nibid/config/config.toml
sed -i -e "s%^address = \"tcp://localhost:1317\"%address = \"tcp://0.0.0.0:13917\"%; s%^address = \":8080\"%address = \":13980\"%; s%^address = \"localhost:9090\"%address = \"0.0.0.0:13990\"%; s%^address = \"localhost:9091\"%address = \"0.0.0.0:13991\"%; s%:8545%:13945%; s%:8546%:13946%; s%:6065%:13965%" $HOME/.nibid/config/app.toml
```

#Download latest chain snapshot
```
curl -L https://snapshots.kjnodes.com/nibiru/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.nibid
[[ -f $HOME/.nibid/data/upgrade-info.json ]] && cp $HOME/.nibid/data/upgrade-info.json $HOME/.nibid/cosmovisor/genesis/upgrade-info.json
```

#Start service and check the logs
```
sudo systemctl start nibiru.service && sudo journalctl -u nibiru.service -f --no-hostname -o cat
```


# 1. Minimum hardware requirement
2 Cores, 8G Ram,  200G SSD, Ubuntu 22.04

# 2.1 Wallet
Add New Wallet Key - Save seed
```
nibid keys add wallet
```
Recover existing key
```
nibid keys add wallet --recover
```
List All Keys
```
nibid keys list
```
# 2.2 Query Wallet Balance
```
nibid q bank balances $(nibid keys show wallet -a)
```
# 2.3 Check sync status
#### False is synced
```
nibid status 2>&1 | jq .sync_info
```
# 2.4 Create Validator
```
nibid tx staking create-validator \
--amount 1000000unibi \
--pubkey $(nibid tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id cataclysm-1 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.05 \
--min-self-delegation 1 \
--from wallet \
--fees 5000unibi \
-y
```




# 3. Backup Validator
#### Important
```
cat $HOME/.nibi/config/priv_validator_key.json
```
# 4. Remove node
```
sudo systemctl stop nibid
sudo systemctl disable nibid
rm -rf /etc/systemd/system/nibid.service
sudo systemctl daemon-reload
sudo rm -f /usr/local/bin/nibid
sudo rm -f $(which nibid)
sudo rm -rf $HOME/.nibid
```

**Upgrade**
```
cd $HOME/nibiru/
git fetch --all
git checkout v1.4.0
make install
nibid version --long | grep -e commit -e version
#version: v1.4.0
#commit: 9ff225a9859731e9547966dbc7c41f23e00d6b36
systemctl restart nibid && journalctl -fu nibid -o cat
#consensus
curl -s http://localhost:26657/consensus_state  | jq '.result.round_state.height_vote_set[0].prevotes_bit_array'
```
