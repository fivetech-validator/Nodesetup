# Nodesetup
SETUP VALIDATOR KYVE

# cài đặt các ứng dụng cần thiết và update ubuntu
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```	

	# CÀI ĐẶT GO
```
ver="1.23.5"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
	#DOWNLOAD VÀ CÀI ĐẶT DỮ LIỆU NODE KYVE
```
cd $HOME
wget https://github.com/KYVENetwork/chain/releases/download/v2.1.0/kyved_mainnet_linux_amd64.tar.gz
tar -xvzf kyved_mainnet_linux_amd64.tar.gz
chmod +x kyved
rm kyved_mainnet_linux_amd64.tar.gz
sudo mv kyved $HOME/go/bin/kyved
```
	# CÀI CẶT NODE

kyved init FIVETECH --chain-id kyve-1
kyved config chain-id kyve-1
```

	#CÁC LỆNH VỀ VÍ

#TẠO VÍ MỚI
```
kyved keys add wallet
```

#RECOVER VÍ CŨ
```
kyved keys add wallet --recover
```

	#DOWNLOAD FILE GENESIS

```
curl https://raw.githubusercontent.com/KYVENetwork/networks/main/kyve-1/genesis.json > ~/.kyve/config/genesis.json
```

	#CÀI ĐẶT  minimum gas price and Peers/Seeds/Filter peers/MaxPeers
```
sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $HOME/.kyve/config/config.toml
seeds=""
peers="b950b6b08f7a6d5c3e068fcd263802b336ffe047@18.198.182.214:26656,25da6253fc8740893277630461eb34c2e4daf545@3.76.244.30:26656"
sed -i.bak -e "s/^seeds *=.*/seeds = \"$seeds\"/; s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" ~/.kyve/config/config.toml
sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 50/g' $HOME/.kyve/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 50/g' $HOME/.kyve/config/config.toml
```
	#DOWNLOAD ADDRBOOK
```
wget -O $HOME/.kyve/config/addrbook.json "https://raw.githubusercontent.com/nodersteam/cosmos-adrbook/main/kyve/addrbook.json"
```

	#TẠO FILE DỮ LIỆU (Create a service file)

```
sudo tee /etc/systemd/system/kyved.service > /dev/null <<EOF
[Unit]
Description=kyve
After=network-online.target

[Service]
User=$USER
ExecStart=$(which kyved) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```


	#SỬ DỤNG STATESYNC(Use our StateSync)

```
sudo systemctl stop kyved

SNAP_RPC=http://kyve.rpc.nodersteam.com:23657
cp $HOME/.kyve/data/priv_validator_state.json $HOME/.kyve/priv_validator_state.json.backup
sed -i -e "/seeds =/ s/= .*/= \"$SEEDS\"/"  $HOME/.kyve/config/config.toml
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 100)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.kyve/config/config.toml
```

	#BẮT ĐẦU CHẠY 1 NODE
```
sudo systemctl daemon-reload
sudo systemctl enable kyved
sudo systemctl restart kyved
sudo journalctl -u kyved -f -o cat
```

	#TẠO THÔNG TIN 1 VALIDATOR
```
kyved tx staking create-validator \
--amount 1000000ukyve \
--moniker="FIVETECH" \
--identity="495CADCFB1CC4C00" \
--website="https://linktr.ee/fivetech_validator" \
--details="CRYPTO VN NO1" \
--commission-rate "0.05" \
--commission-max-rate "0.20" \
--commission-max-change-rate "0.05" \
--min-self-delegation "1" \
--pubkey "$(kyved tendermint show-validator)" \
--from wallet \
--gas 51000000 \
--fees 1020000ukyve \
--chain-id kyve-1
```


	#edit
```
kyved tx staking edit-validator \
--new-moniker="FIVETECH" \
--identity="495CADCFB1CC4C00" \
--details="CRYPTO VN NO1" \
--website="https://linktr.ee/fivetech_validator" \
--chain-id kyve-1 \
--commission-rate 0.05 \
--from wallet \
--fees 1020000ukyve \
-y
```
	#XÓA NODE
```
sudo systemctl stop kyved
sudo systemctl disable kyved
rm /etc/systemd/system/kyved.service
sudo systemctl daemon-reload
cd $HOME
rm -rf chain
rm -rf .kyve
rm -rf $(which kyved)
```

        #stake kyve
```
kyved tx staking delegate $(kyved keys show wallet --bech val -a) 80000000000ukyve --from wallet --chain-id kyve-1 --fees 1020000ukyve -y
```

thoat tu
```
kyved tx slashing unjail --from wallet --chain-id kyve-1 --gas 350000 --fees 975000ukyve -y

```
claim rw vs commission
```
kyved tx distribution withdraw-rewards $(kyved keys show wallet --bech val -a) --commission --from wallet --chain-id kyve-1 --fees 1020000ukyve -y
```

