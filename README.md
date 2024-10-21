#SETUP NODE SUORCE


# install dependencies, if needed
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

# install go, if needed
```
cd $HOME
VER="1.22.1"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

# set vars
```
echo "export WALLET="wallte"" >> $HOME/.bash_profile
echo "export MONIKER="FIVETECH"" >> $HOME/.bash_profile
echo "export SOURCE_CHAIN_ID="source-1"" >> $HOME/.bash_profile
echo "export SOURCE_PORT="32"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

# download binary
```
cd $HOME
rm -rf source
git clone https://github.com/Source-Protocol-Cosmos/source.git
cd source
git checkout v3.0.3
make install
```

# config and init app
```
sourced config node tcp://localhost:${SOURCE_PORT}657
sourced config keyring-backend os
sourced config chain-id source-1
sourced init "FIVETECH" --chain-id source-1
```

# download genesis and addrbook
```
wget -O $HOME/.source/config/genesis.json https://mainnet-files.itrocket.net/source/genesis.json
wget -O $HOME/.source/config/addrbook.json https://mainnet-files.itrocket.net/source/addrbook.json
```

# set seeds and peers
```
SEEDS="7347b05f140e4ed5d3da7b26c754a486dc1d2ecd@source-mainnet-seed.itrocket.net:32656"
PEERS="8a812024b8a5b4539878b03ac2f822655831ca5f@source-mainnet-peer.itrocket.net:32656,29aaebc5b17674c3529ac1a4a4d040824aba64fa@54.202.237.247:26656,0107ac60e43f3b3d395fea706cb54877a3241d21@35.87.85.162:26656,901d8400432c6ab22ce1367681e7ac2fb1b11a77@5.75.245.174:26656,c223adbf2ba594cb6254a82fff92c00d14bca5c2@147.182.211.27:26656,96d63849a529a15f037a28c276ea6e3ac2449695@34.222.1.252:26656,637077d431f618181597706810a65c826524fd74@135.181.0.47:15856,7fdcff8e08f34618fbf445d3f0d90df15e6f58f5@194.60.201.69:26656,7347b05f140e4ed5d3da7b26c754a486dc1d2ecd@142.132.253.112:32656,66d0bdbf3f427bf840d56cc6c8ece71667637999@5.9.147.22:26356"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.source/config/config.toml
```

# set custom ports in app.toml
```
sed -i.bak -e "s%:1317%:${SOURCE_PORT}317%g;
s%:8080%:${SOURCE_PORT}080%g;
s%:9090%:${SOURCE_PORT}090%g;
s%:9091%:${SOURCE_PORT}091%g;
s%:8545%:${SOURCE_PORT}545%g;
s%:8546%:${SOURCE_PORT}546%g;
s%:6065%:${SOURCE_PORT}065%g" $HOME/.source/config/app.toml
```


# set custom ports in config.toml file
```
sed -i.bak -e "s%:26658%:${SOURCE_PORT}658%g;
s%:26657%:${SOURCE_PORT}657%g;
s%:6060%:${SOURCE_PORT}060%g;
s%:26656%:${SOURCE_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${SOURCE_PORT}656\"%;
s%:26660%:${SOURCE_PORT}660%g" $HOME/.source/config/config.toml
```

# config pruning
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.source/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.source/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.source/config/app.toml
```

# set minimum gas price, enable prometheus and disable indexing
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.25usource"|g' $HOME/.source/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.source/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.source/config/config.toml
```

# create service file
```
sudo tee /etc/systemd/system/sourced.service > /dev/null <<EOF
[Unit]
Description=Source node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.source
ExecStart=$(which sourced) start --home $HOME/.source
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

# reset and download snapshot
```
sourced tendermint unsafe-reset-all --home $HOME/.source
if curl -s --head curl https://mainnet-files.itrocket.net/source/snap_source.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://mainnet-files.itrocket.net/source/snap_source.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.source
    else
  echo no have snap
fi
```

# enable and start service
```
sudo systemctl daemon-reload
sudo systemctl enable sourced
sudo systemctl restart sourced && sudo journalctl -u sourced -f
```

# to create a new wallet, use the following command. don’t forget to save the mnemonic
```
sourced keys add wallet
```

# to restore exexuting wallet, use the following command
```
sourced keys add wallet --recover
```

# save wallet and validator address
```
WALLET_ADDRESS=$(sourced keys show $WALLET -a)
VALOPER_ADDRESS=$(sourced keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

# check sync status, once your node is fully synced, the output from above will print "false"
```
sourced status 2>&1 | jq 
```

# before creating a validator, you need to fund your wallet and check balance
```
sourced query bank balances $WALLET_ADDRESS 
```
#create node
```
sourced tx staking create-validator \
--amount 1000000usource \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(sourced tendermint show-validator) \
--moniker "FIVETECH" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id source-1 \
--gas auto --gas-adjustment 1.5 --fees=90000usource \
-y
```


#edit node
```
sourced tx staking edit-validator \
--new-moniker "FIVETECH" \
--identity "495CADCFB1CC4C00" \
--details "Blockchain | Node & Validator" \
--website "https://explorer.fivetech.pro/" \
--security-contact "phamvannamdc@gmail.com" \
--chain-id source-1 \
--commission-rate 0.05 \
--from wallet \
--gas auto --gas-adjustment 1.5 --fees=90000usource \
-y 
```





#delete node
```
sudo systemctl stop sourced
sudo systemctl disable sourced
sudo rm -rf /etc/systemd/system/sourced.service
sudo rm $(which sourced)
sudo rm -rf $HOME/.source
sed -i "/SOURCE_/d" $HOME/.bash_profile
```


Service operations ⚙️
Check logs
```
sudo journalctl -u sourced -f
```
Start service
```
sudo systemctl start sourced
```
Stop service
```
sudo systemctl stop sourced
```
Restart service
```
sudo systemctl restart sourced
```
Check service status
```
sudo systemctl status sourced
```
Reload services
```
sudo systemctl daemon-reload
```
Enable Service
```
sudo systemctl enable sourced
```
Disable Service
```
sudo systemctl disable sourced
```

Node info
```
sourced status 2>&1 | jq
```

Unjail
```
sourced tx slashing unjail --from wallet --chain-id source-1 --gas 350000 --fees "97500"usource -y
```

claim phí node
```
sourced tx distribution withdraw-rewards sourcevaloper130wjchtwnx49utu7d8jkpw28seq23sy7yz07eh --from wallet --commission --chain-id source-1 --gas auto --gas-adjustment 1.5 --fees=50000usource -y 
```
