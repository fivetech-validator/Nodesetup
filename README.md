# Nodesetup
Installation Node

Network Type: Mainnet
Chain-id: medasdigital-2
Current Node version: 1.0.0
Hardware requirements
The following requirements are recommended for running Defund:

4 or more physical CPU cores
At least 200GB of SSD disk storage
At least 8GB of memory (RAM)
At least 100mbps network bandwidth
Install dependencies Required
```
sudo apt update && sudo apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```
Install go
We are gonna use GO Version 1.23.3 If you already have GO installed you can skip this

```
ver="1.23.3"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
source ~/.bash_profile
go version
```
Install Cosmwasm
```
rm /usr/lib/libwasmvm.x86_64.so
wget -P /usr/lib https://github.com/CosmWasm/wasmvm/releases/download/v2.1.2/libwasmvm.x86_64.so
sudo ldconfig
```
Install binary
```
cd $HOME
wget https://github.com/oxygene76/medasdigital-2/releases/download/v1.0.0/medasdigitald
chmod +x medasdigitald
mv medasdigitald $HOME/go/bin/
Init Change <MONIKER> with ur Name
medasdigitald init FIVETECH --chain-id medasdigital-2
```
Download Genesis file and addrbook
Genesis
```
wget -O $HOME/.medasdigital/config/genesis.json https://raw.githubusercontent.com/oxygene76/medasdigital-2/refs/heads/main/genesis/mainnet/config/genesis.json
```
Addrbook
```
wget -O $HOME/.medasdigital/config/addrbook.json https://raw.githubusercontent.com/vinjan23/Mainnet/refs/heads/main/Medas/addrbook.json
```
Configure Peers & Gas
```
peers="51ca3b0a3663af88566b32ecfd77948e55000bcc@88.205.101.195:26656,90be2e9f0a279372d2931e38f15025db9a847dbd@88.205.101.196:26656,0e567c9efe6e6d15f9b3257679398368c2ab04bb@88.205.101.197:26656,669d1b9f9c4bb99df594abaee4b13ae1b14d37a6@64.251.18.192:26656,cbfcd111ee19483dbbfed0919ac0d23119c5f0fe@67.207.180.166:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.medasdigital/config/config.toml
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025umedas\"|" $HOME/.medasdigital/config/app.toml
```
Config pruning
```
sed -i \
-e 's|^pruning *=.*|pruning = "custom"|' \
-e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
-e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
-e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
$HOME/.medasdigital/config/app.toml
```
Indexer Off
```
sed -i 's|^indexer *=.*|indexer = "null"|' $HOME/.medasdigital/config/config.toml
```
create service file and start node
```
sudo tee /etc/systemd/system/medasdigitald.service > /dev/null << EOF
[Unit]
Description=medasdigital
After=network-online.target
[Service]
User=$USER
ExecStart=$(which medasdigitald) start
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
Enable Service and Start Node
```
sudo systemctl daemon-reload
sudo systemctl enable medasdigitald
sudo systemctl restart medasdigitald
sudo journalctl -u medasdigitald -f -o cat
```

Wallet Management
Add Wallet Specify the value <wallet> with your own wallet name
```
medasdigitald keys add wallet
```
Recover Wallet
```
medasdigitald keys add wallet --recover
```
List Wallet
```
medasdigitald keys list
```
Delete Wallet
```
medasdigitald keys delete wallet
```
Check Wallet Balance
```
medasdigitald  q bank balances $(medasdigitald keys show wallet -a)
```
Validator Management
Please adjust <wallet> , MONIKER , YOUR_KEYBASE_ID , YOUR_DETAILS , YOUUR_WEBSITE_URL

Create Validator (Staking)
```
medasdigitald tendermint show-validator
```

```
nano /root/.medasdigital/validator.json
```

```
{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"......"},
  "amount": "115000000umedas",
  "moniker": "",
  "identity": "",
  "website": "https://service.vinjan.xyz",
  "security": "",
  "details": "Staking Provider-IBC Relayer",
  "commission-rate": "0.05",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.05",
  "min-self-delegation": "1"
}
```
```
medasdigitald tx staking create-validator $HOME/.medasdigital/validator.json \
--from wallet \
--chain-id medasdigital-2 \
--gas-prices 0.025umedas \
--gas-adjustment 1.5 \
--gas auto
```

Edit Validator
```
medasdigitald tx staking edit-validator \
--new-moniker="Vinjan.Inc" \
--identity="7C66E36EA2B71F68" \
--commission-rate="0.1" \
--chain-id=medasdigital-2 \
--from=wallet \
--gas-prices 0.025umedas \
--gas-adjustment 1.5 \
--gas auto
```
Unjail Validator
```
medasdigitald tx slashing unjail --from wallet --chain-id medasdigital-2
```
Check Jailed Reason
```
medasdigitald query slashing signing-info $(medasdigitald tendermint show-validator)
```
Token Management
Withdraw Rewards
```
medasdigitald tx distribution withdraw-all-rewards --from wallet --chain-id medasdigital-2 --gas-adjustment 1.5 --gas-prices 0.025umedas  --gas auto
```
Withdraw Rewards with Comission
```
medasdigitald tx staking delegate $(medasdigitald keys show wallet --bech val -a) --commission --from wallet --chain-id medasdigital-2 --gas-adjustment 1.5 --gas-prices 0.025umedas  --gas auto
```
Delegate Token to your own validator
```
medasdigitald tx staking delegate $(medasdigitald keys show wallet --bech val -a) 1000000umedas --from wallet --chain-id medasdigital-2 --gas-adjustment 1.5 --gas-prices 0.025umedas  --gas auto
```
Delegate Token to other validator
```
medasdigitald tx staking redelegate $(medasdigitald keys show wallet --bech val -a) <TO_VALOPER_ADDRESS> 1000000umedas --from wallet ---chain-id medasdigital-2 --gas-adjustment 1.5 --gas-prices 0.025umedas  --gas auto
```
Unbond Token from your validator
```
medasdigitald tx staking unbond $(medasdigitald keys show wallet --bech val -a) 1000000umedas --from wallet --chain-id medasdigital-2 --gas-adjustment 1.5 --gas-prices 0.025umedas  --gas auto
```
Send Token to another wallet
```
medasdigitald tx bank send wallet <TO_WALLET_ADDRESS> 1000000umeads --from wallet --chain-id medasdigital-2 --gas-adjustment 1.5 --gas-prices 0.025umedas  --gas auto
```
Governance
Vote You can change the value of yes to no,abstain,nowithveto
```
medasdigitald tx gov vote 2 yes --from wallet --chain-id medasdigital-2
```
Other
Set Your own Custom Ports You can change value CUSTOM_PORT=44 To any other ports

PORT=23
```
sed -i.bak -e  "s%^node = \"tcp://localhost:26657\"%node = \"tcp://localhost:23657\"%" $HOME/.medasdigital/config/client.toml
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:23658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:23657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:23060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:23656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":23660\"%" $HOME/.medasdigital/config/config.toml
sed -i.bak -e "s%^address = \"tcp://localhost:1317\"%address = \"tcp://localhost:23317\"%; s%^address = \"localhost:9090\"%address = \"localhost:23090\"%" $HOME/.medasdigital/config/app.toml
```
Enable Indexing usually enabled by default
```
sed -i 's|^indexer *=.*|indexer = "kv"|' $HOME/.medasdigital/config/config.toml
```
Disable Indexing
```
sed -i 's|^indexer *=.*|indexer = "null"|' $HOME/.medasdigital/config/config.toml
```
Reset Chain Data
```
medasdigitald tendermint unsafe-reset-all --home $HOME/.medasdigital --keep-addr-book
```
Delete Node
WARNING! Use this command wisely Backup your key first it will remove Defund from your system
```
sudo systemctl stop medasdigitald
sudo systemctl disable medasdigitald
sudo rm /etc/systemd/system/medasdigitald.service
sudo systemctl daemon-reload
rm -f $(which medasdigitald)
rm -rf .medasdigital
rm -rf medasdigital
```
