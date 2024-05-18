# initia

### Replace YOUR_MONIKER_GOES_HERE with your validator name
```MONIKER="YOUR_MONIKER_GOES_HERE"```


### Install dependencies
### UPDATE SYSTEM AND INSTALL BUILD TOOLS
```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```

### INSTALL GO
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.21.10.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

### Download and build binaries
# Clone project repository
```
cd $HOME
rm -rf initia
git clone https://github.com/initia-labs/initia.git
cd initia
git checkout v0.2.14
```

# Build binaries
```
make build
```

# Prepare binaries for Cosmovisor
```
mkdir -p $HOME/.initia/cosmovisor/genesis/bin
mv build/initiad $HOME/.initia/cosmovisor/genesis/bin/
rm -rf build
```

# Create application symlinks
```
sudo ln -s $HOME/.initia/cosmovisor/genesis $HOME/.initia/cosmovisor/current -f
sudo ln -s $HOME/.initia/cosmovisor/current/bin/initiad /usr/local/bin/initiad -f
```

### Install Cosmovisor and create a service
# Download and install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

# Create service
```
sudo tee /etc/systemd/system/initia.service > /dev/null << EOF
[Unit]
Description=initia node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.initia"
Environment="DAEMON_NAME=initiad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.initia/cosmovisor/current/bin"
```

```
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable initia.service
```

### Initialize the node
# Set node configuration
```
initiad config set client chain-id initiation-1
initiad config set client keyring-backend test
initiad config set client node tcp://localhost:17957
```

# Initialize the node
```
initiad init $MONIKER --chain-id initiation-1
```

# Download genesis and addrbook from kjnodes
```
curl -Ls https://snapshots.kjnodes.com/initia-testnet/genesis.json > $HOME/.initia/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/initia-testnet/addrbook.json > $HOME/.initia/config/addrbook.json
```

# Add seeds
```
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@initia-testnet.rpc.kjnodes.com:17959\"|" $HOME/.initia/config/config.toml
```

# Set minimum gas price
```
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.15uinit,0.01uusdc\"|" $HOME/.initia/config/app.toml
```

# Set pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.initia/config/app.toml
```

# Set custom ports
```
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:17958\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:17957\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:17960\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:17956\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":17966\"%" $HOME/.initia/config/config.toml
sed -i -e "s%^address = \"tcp://localhost:1317\"%address = \"tcp://0.0.0.0:17917\"%; s%^address = \":8080\"%address = \":17980\"%; s%^address = \"localhost:9090\"%address = \"0.0.0.0:17990\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:17991\"%; s%:8545%:17945%; s%:8546%:17946%; s%:6065%:17965%" $HOME/.initia/config/app.toml
```

### Download latest chain snapshot
```
curl -L https://snapshots.kjnodes.com/initia-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.initia
[[ -f $HOME/.initia/data/upgrade-info.json ]] && cp $HOME/.initia/data/upgrade-info.json $HOME/.initia/cosmovisor/genesis/upgrade-info.json
```

### Start service and check the logs
```
sudo systemctl start initia.service && sudo journalctl -u initia.service -f --no-hostname -o cat
```

### Set up validator
Official documentation: https://docs.initia.xyz/run-initia-node/running-initia-node/becoming-a-validator

### 1. Create a wallet
First of all we will need to create wallet for our validator. You have two options for that.

### OPTION 1 - CREATE NEW WALLET
```
initiad keys add wallet
```

### OPTION 2 - RECOVER EXISTING WALLET
```
initiad keys add wallet --recover
```
### Save the mnemonic output as this is the only way to recover your validator wallet in case you lose it!

To list your wallets use command below
```
initiad keys list
```

### 2. Fund a wallet
To create validator you have to fund the previously created wallet using https://faucet.testnet.initia.xyz/

To check wallet balance use command below
```
initiad q bank balances $(initiad keys show wallet -a)
```

### 3. Create validator
Please make sure you have adjusted moniker, identity, details, website to match your values.
```
initiad tx mstaking create-validator \
--amount 1000000uinit \
--pubkey $(initiad tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id initiation-1 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.05 \
--from wallet \
--fees 5000uinit \
-y
```

### Save the ~/.initia/config/priv_validator_key.json file as this is the only way to recover your validator signing key in case you lose it!

### Set up Oracle
This guide is only for validator nodes

Official documentation: https://docs.initia.xyz/run-initia-node/running-initia-node/oracle

The Slinky Oracle consists of two main elements:

An on-chain component that retrieves price data from the sidecar with each block, forwards these prices to the blockchain through vote extensions, and compiles prices from all validators involved.

A sidecar process dedicated to polling price information from various providers and delivering this data to the on-chain component.

Step 1: Clone the Repository and build binaries

# Clone repository
```
cd $HOME
rm -rf slinky
git clone https://github.com/skip-mev/slinky.git
cd slinky
git checkout v0.4.3
```

# Build binaries
```
make build
```

# Move binary to local bin
```
mv build/slinky /usr/local/bin/
rm -rf build
```

### Step 2: Run oracle
CREATE SYSTEMD SERVICE
```
sudo tee /etc/systemd/system/slinky.service > /dev/null <<EOF
[Unit]
Description=Initia Slinky Oracle
After=network-online.target

[Service]
User=$USER
ExecStart=$(which slinky) --oracle-config-path $HOME/slinky/config/core/oracle.json --market-map-endpoint 127.0.0.1:17990
Restart=on-failure
RestartSec=30
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### ENABLE AND START SYSTEMD SERVICE
```
sudo systemctl daemon-reload
sudo systemctl enable slinky.service
sudo systemctl start slinky.service
```

### Step 3: Validating Prices
Upon launching the oracle, you should observe successful price retrieval from the provider sources. Additionally, you have the option to execute the test client script available in the Slinky repository by using the command:

```
make run-oracle-client
```

### Step 4: Enable Oracle Vote Extension
In order to utilize the Slinky oracle data in the Initia node, the Oracle setting should be enabled in the config/app.toml file.
```
###############################################################################
###                                  Oracle                                 ###
###############################################################################
[oracle]
# Enabled indicates whether the oracle is enabled.
enabled = "true"

# Oracle Address is the URL of the out of process oracle sidecar. This is used to
# connect to the oracle sidecar when the application boots up. Note that the address
# can be modified at any point, but will only take effect after the application is
# restarted. This can be the address of an oracle container running on the same
# machine or a remote machine.
oracle_address = "127.0.0.1:8080"

# Client Timeout is the time that the client is willing to wait for responses from 
# the oracle before timing out.
client_timeout = "500ms"

# MetricsEnabled determines whether oracle metrics are enabled. Specifically
# this enables instrumentation of the oracle client and the interaction between
# the oracle and the app.
metrics_enabled = "false"
```


### Step 5: Check the systemd logs
To check service logs use command below:

```
journalctl -fu slinky --no-hostname
```

Successfull Log examples:
```
14T19:07:08.296Z","num_prices":65}
May 14 19:07:08 slinky[877177]: {"level":"info","ts":"2024-05-14T19:07:08.547Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":877177,"process":"oracle","last_sync":"2024-05-14T19:07:08.547Z","num_prices":65}
May 14 19:07:08 slinky[877177]: {"level":"info","ts":"2024-05-14T19:07:08.796Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":877177,"process":"oracle","last_sync":"2024-05-14T19:07:08.796Z","num_prices":65}
May 14 19:07:09 slinky[877177]: {"level":"info","ts":"2024-05-14T19:07:09.045Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":877177,"process":"oracle","last_sync":"2024-05-14T19:07:09.045Z","num_prices":65}
May 14 19:07:09 slinky[877177]: {"level":"info","ts":"2024-05-14T19:07:09.296Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":877177,"process":"oracle","last_sync":"2024-05-14T19:07:09.296Z","num_prices":65}
May 14 19:07:09 slinky[877177]: {"level":"info","ts":"2024-05-14T19:07:09.544Z","caller":"marketmap/fetcher.go:116","msg":"successfully fetched market map data from
```
