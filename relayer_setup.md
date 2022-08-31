
# Setup GO Relayer v2 between Stride and GAIA
In current example we will learn how to set up GO Relayer v2 between two cosmos chains

## Preparation before you start
Before setting up relayer you need to make sure you already have:
1. Fully synchronized RPC nodes for each Cosmos project you want to connect
2. RPC enpoints should be exposed and available from relayer instance

Or optionally you can use an available RPC endpoints.

#### RPC configuration is located in `config.toml` file
```
# STRIDE
laddr = "tcp://0.0.0.0:16657" in $HOME/.stride/config/config.toml
# GAIA
laddr = "tcp://0.0.0.0:23657" in $HOME/.gaia/config/config.toml  
```

3. Indexing is set to `kv` and is enabled on each node
```
# STRIDE
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.stride/config/config.toml
# GAIA
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.gaia/config/config.toml  
```

4. Dont forget to restart fullnode service after making changes
5. For each chain you will need to have wallets that are funded with tokens. This wallets will be used to do all relayer stuff

## Set up variables
All settings below are just example for IBC Relayer between `STRIDE-TESTNET-4` and `GAIA` testnets. Please fill with your own values.
```
RELAYER_ID='openbitlab#9650' # add your Discord username here
STRIDE_RPC_ADDR='127.0.0.1:16657'    # use your own or an available Stride RPC enpoint here
GAIA_RPC_ADDR='127.0.0.1:23657'      # use your own or an available Gaia RPC enpoint here
```

## Update system
```
sudo apt update && sudo apt upgrade -y
```

## Install GO
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.18.3"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```

## Install and initialize GO Relayer v2
```
git clone https://github.com/cosmos/relayer.git
cd relayer && git checkout v2.0.0-rc4
make install
rly config init --memo $RELAYER_ID
sudo mkdir $HOME/.relayer/chains
sudo mkdir $HOME/.relayer/paths
```

## Create relayer configuration files
### 1. Generate CHAIN_A config file using variables we have defined above
```
sudo tee $HOME/.relayer/stride.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "STRIDE-TESTNET-4",
    "rpc-addr": "http://${STRIDE_RPC_ADDR}",
    "account-prefix": "stride",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.000ustrd",
    "gas": 200000,
    "trusting-period": "8h",
    "timeout": "20s",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

### 2. Generate CHAIN_B config file using variables we have defined above
```
sudo tee $HOME/.relayer/gaia.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "GAIA",
    "rpc-addr": "http://${GAIA_RPC_ADDR}",
    "account-prefix": "cosmos",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.000uatom",
    "gas": 200000,
    "trusting-period": "8h",
    "timeout": "20s",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

## Load chain configuration into the relayer
```
rly chains add --file=$HOME/.relayer/chains/stride.json stride
rly chains add --file=$HOME/.relayer/chains/gaia.json gaia
```

Check chains added to relayer
```
rly chains list
```

Successful output:
```
1: GAIA             -> type(cosmos) key(✘) bal(✘) path(✘)
2: STRIDE-TESTNET-4 -> type(cosmos) key(✘) bal(✘) path(✘)
```

## Load wallets into the relayer
```
rly keys restore stride wallet "your stride wallet mnemonic goes here"
rly keys restore gaia wallet "your gaia wallet mnemonic goes here"
```

Check wallet balance
```
rly q balance stride
rly q balance gaia
```

## Add path for STRIDE-TESTNET-4 and GAIA to relayer configuration file
Open `$HOME/.relayer/config/config.yaml` file and replace `paths: {}` with:
```
paths:
    stride-gaia:
        src:
            chain-id: STRIDE-TESTNET-4
            client-id: 07-tendermint-0
            connection-id: connection-0
        dst:
            chain-id: GAIA
            client-id: 07-tendermint-0
            connection-id: connection-0
        src-channel-filter:
            rule: allowlist
            channel-list: 
             - channel-0
             - channel-1
             - channel-2
             - channel-3
             - channel-4 
```

Check path is correct
```
rly paths list
```

Successful output:
```
0: stride-gaia -> chns(✔) clnts(✔) conn(✔) (STRIDE-TESTNET-4<>GAIA)
```
If everything is correct, we can proceed with relayer service creation

## Create service
```
sudo tee /etc/systemd/system/relayerd.service > /dev/null <<EOF
[Unit]
Description=Stride zone relayer
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
ExecStart=$(which rly) start stride-gaia -p events
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Start the service
```
sudo systemctl daemon-reload
systemctl enable relayerd
systemctl start relayerd
```

## Check logs
```
journalctl -u relayerd -f -o cat
```

Check your wallet transaction in explorer to find the GO v2 Relayer transactions:

![image](https://user-images.githubusercontent.com/35404870/187739467-a5289447-0198-4810-b4f5-4888bb8878e5.png)
