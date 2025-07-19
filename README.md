# Nodesetup
SETUP DUNGONE CHAIN
 GITHUP: https://github.com/Crypto-Dungeon/dungeonchain/tree/main/network/dungeon-1

 Update system and install build tools:
```
sudo apt update && sudo apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc chrony liblz4-tool -y
```


Install Go:
```
ver="1.23.0"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

Install node:


```
cd $HOME && mkdir -p go/bin/
git clone https://github.com/Crypto-Dungeon/dungeonchain.git
cd dungeonchain
git fetch --tags
git checkout v5.0.0
make install
dungeond version
```

update
```
cd $HOME && mkdir -p go/bin/
rm -rf dungeonchain
git clone https://github.com/Crypto-Dungeon/dungeonchain.git
cd dungeonchain
git fetch --tags
git checkout v5.0.0
make install
dungeond version
```

#Download and build binaries
```
cd $HOME

rm -rf dungeonchain

git clone git clone https://github.com/Crypto-Dungeon/dungeonchain.git
cd dungeonchain

cd dungeonchain

git fetch --all

git checkout main

make install

dungeond version
```


Initialize Node

```
dungeond init fivetech --chain-id=dungeon-1
dungeond config chain-id dungeon-1
```



Download  Addrbook and Genesis:
```
wget -O $HOME/.dungeonchain/config/addrbook.json "https://snapshots.whenmoonwhenlambo.money/dungeon-1/addrbook.json"
wget -O $HOME/.dungeonchain/config/genesis.json "https://snapshots.whenmoonwhenlambo.money/dungeon-1/genesis.json"
```


Add peers:
```
SEEDS="2a0a3fbbf3c1f8ee159c475c0b24e9869388960a@159.69.81.109:26656"
PEERS=f174206d6b3dbc2dd17cdd884bdfc6ad37268a09@67.218.8.88:26665,5545dc6fa6537ce464a49593bac02258fd963e57@67.218.8.88:26665,cd2311ffdae014daff80c343c26a393e714c7973@172.31.23.120:26656
sed -i.bak -e "s/^persistent*peers *=.\_/persistent_peers = \"$PEERS\"/" $HOME/.dungeonchain/config/config.toml


sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.dungeonchain/config/app.toml


sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0udgn\"|" $HOME/.dungeonchain/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.dungeonchain/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.dungeonchain/config/config.toml
```


Create a service file:


```
sudo tee /etc/systemd/system/dungeond.service > /dev/null <<EOF
[Unit]
Description=dungeond Daemon
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.dungeonchain
ExecStart=$(which dungeond) start --home $HOME/.dungeonchain
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

SnapShot Mainnet:

```
sudo apt install lz4 -y
sudo systemctl stop dungeond
cp $HOME/.dungeonchain/data/priv_validator_state.json $HOME/.dungeonchain/priv_validator_state.json.backup
rm -rf $HOME/.dungeonchain/data
curl -L https://snapshots.whenmoonwhenlambo.money/dungeon-1/dungeon-1-snapshot-latest.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.dungeonchain
mv $HOME/.dungeonchain/priv_validator_state.json.backup $HOME/.dungeonchain/data/priv_validator_state.json
sudo systemctl restart dungeond
sudo journalctl -u dungeond -f -o cat
```

Enable Service and Start Node:
```
sudo systemctl daemon-reload
sudo systemctl enable dungeond
sudo systemctl restart dungeond
sudo journalctl -u dungeond -f -o cat
```


tao vi:
```
dungeond keys add wallet
```
backup vi
```
dungeond keys add wallet --recover
```


Create Validator (Staking)

```
dungeond tendermint show-validator
```





```
nano /root/.dungeonchain/validator.json
```


```

{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"nSyo1A58A60cLmGfzjeOJG1l8jUqFtIeCn9eLWwJw8o="},
  "amount": "1000000udgn",
  "moniker": "FIVETECH",
  "identity": "495CADCFB1CC4C00",
  "website": "https://explorer.fivetech.pro/",
  "security": "https://t.me/fivetech_validator",
  "details": "Blockchain | Node & Validator",
  "commission-rate": "0.05",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.05",
  "min-self-delegation": "1"
}

```


```
dungeond tx staking create-validator $HOME/.dungeonchain/validator.json \
--from wallet \
--chain-id dungeon-1 \
--gas-prices 0.025udgn \
--gas-adjustment 1.5 \
--gas auto
```

Unjail Validator
```
dungeond tx slashing unjail --from wallet --chain-id dungeon-1 --gas 350000 --fees "97500"udgn -y
```

Withdraw all rewards from all validators

```
dungeond tx distribution withdraw-all-rewards --from wallet --chain-id dungeon-1 --gas 350000 -y
```

Withdraw and commission from your Validator
```
dungeond tx distribution withdraw-rewards dungeonvaloper14ka9c4pdyz8kchkfvz6ae0guvcz0rfk4yntsjw --from wallet --gas 350000 --chain-id=dungeon-1 --commission -y
```



