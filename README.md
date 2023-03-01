# ** DEPRECATED **

Code was written before the merge.
To be compatible for the merge, changes must be made.

# Guide to Staking on Ethereum 2.0 (Ubuntu, Prysm)

# Overview

Here’s a super-simplified diagram to help you conceptualize what we are going to do. The yellow boxes are the parts this guide mostly covers.

![Alt-Tex](/images/overview.png)

The conceptual flow is:

- Set up a Eth1 node and sync it with the Eth1 Göerli testnet (Or use third party provider)
- Configure Beacon Node and sync it with the Eth1 Node
- Generate and activate validator keys
- Configure the Validator Client
- The Beacon Node makes the magic happen (blocks, attestations, slashings) with the help of the validator (signing).

# Testnet (Pyrmont)

This is a step-by-step guide to staking on Ethereum 2.0 via the Pyrmont multi-client testnet. It is based on the following technologies

- Ubuntu v20.04 (LTS) x64 server
- Go Ethereum Node (code branch)
- Prysmatic Labs Ethereum 2.0 client — Prysm (code branch)
- Official multi-client testnet public network, Pyrmont
- MetaMask crypto wallet browser extension
- Prometheus metrics
- Grafana dashboard

This guide includes instructions on how to:

- Configure a newly running Ubuntu server instance
- Configure and run an Ethereum 1.0 node as a service
- Generate a Prysm wallet and import Pyrmont validator account keys
- Compile and configure the Prysmatic Labs beacon chain and validator client software for Ethereum 2.0,Phase 0 (Pyrmont testnet) and run them as a service
- Install and configure Prometheus metrics and set up a Grafana dashboard

## Prerequisites

This guide assumes some knowledge of Ethereum, ETH, staking, Linux, and MetaMask.
This guide also requires the following are installed and running before getting started:

- Ubuntu server v20.04 (LTS) amd64 or newer, installed and running on a local computer on your network or in the cloud (AWS, Digital Ocean, Microsoft Azure, etc.). A local computer is recommended for greater decentralization — if the cloud provider goes down then all nodes hosted with that provider go down.
- MetaMask crypto wallet browser extension, installed and configured. A computer with a desktop (Mac, Windows, Linux) and a browser (Safari, Brave, FireFox, etc.) is required.

## Requirements

- Ubuntu server instance. I used v20.04 (LTS) amd64 server VM.
- MetaMask crypto wallet browser extension, installed and configured.
- Hardware recommended requirements to run Prysm software:
  - Processor: Intel Core i7–4770 or AMD FX-8310 or better
  - Memory: 16GB RAM
  - Storage: 100GB available space SSD (Prysm client only)

NOTE: Hardware requirements are a broad topic. In general a relatively modern CPU, 16GB RAM, a SSD of at least 1TB, and a stable internet connection with sufficient download speed and monthly data allowance are likely required for good staking performanc

Minimum specifications Prysm
These specifications must be met in order to successfully run the Prysm client.

- Operating System: 64-bit Linux, Mac OS X 10.14+, Windows 64-bit
- Processor: Intel Core i5–760 or AMD FX-8100 or better
- Memory: 8GB RAM
- Storage: 20GB available space SSD
- Internet: Broadband connection

Recommended specifications
These hardware specifications are recommended, but not required to run the Prysm client.

- Processor: Intel Core i7–4770 or AMD FX-8310 or better
- Memory: 16GB RAM
- Storage: 100GB available space SSD
- Internet: Broadband connection

## Step 0 — Connect to the Server

Using a SSH client, connect to your Ubuntu server.

```bash
root@eth-node-01.lcx.cloud.css-it.net -p 10001 -i /Users/florolf/Desktop/ETH-Node/eth-node-01/privateKey
```

## Step 1 — Update the Server

Make sure your system is up to date with the latest software and security updates.

```bash
apt update && sudo apt upgrade
apt dist-upgrade && sudo apt autoremove
```

## Step 2 — Secure the Server

### Configure the firewall

Ubuntu 20.04 servers can use the default UFW firewall to restrict inbound traffic to the server. Before we enable it we need to allow inbound traffic for SSH, Go Ethereum, Grafana, and Prysm.

### Allow SSH

Allows connection to the server over SSH. For security reasons we are going to modify the default port of 22 because it is a common attack vector.

### Allow Go Ethereum

Allows incoming requests from Go Ethereum peers (port 30303/TPC and 30303/UDP). If you’d rather use a node hosted by a 3rd party (Infura, etc.) then skip this step.

```bash
sudo ufw allow 30303
```

### Allow Prysm

Allows P2P connections with peers for actions on the beacon node. Ports 13000/TCP and 12000/UDP are listed as defaults by Prysmatic Labs.

Note: If you are hosting your Ubuntu instance locally your internet router and/or firewall will need to be configured to allow incoming traffic on these ports as well.

```bash
 ufw allow 13000/tcp
 ufw allow 12000/udp
```

### Allow Grafana

Allows incoming requests to the Grafana web server (port 3000/TCP).

```bash
ufw allow 3000/tcp
```

### Allow Prometheus (Optional)

If you want direct access to the Prometheus data service you can open up port 9090/TCP as well. This is not necessary if you are solely using Grafana to view the data. I did not open this port.

```bash
ufw allow 9090/tcp
```

Enable the firewall and check to verify the rules have been correctly configured.

```bash
ufw enable
ufw status numbered
```

Output should look something like this:

![Alt-Tex](/images/ufw.png)

Set up an Ethereum (Eth1) Node
An Ethereum node is required for staking.

https://infura.io/

Testnet Endpoint:
https://goerli.infura.io/v3/abdecbed7270438cbfe704cdc26424cc

## Step 3: Set up an Ethereum (Eth2) Client Prysm

Open a terminal in the desired directory for Prysm. Then create a working directory and enter it:

```bash
mkdir prysm && cd prysm
```

Fetch the prysm.sh script from Github and make it executable:

```bash
curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output prysm.sh && chmod +x prysm.sh
```

### Configure the Prysm Beacon Node

configure and run the beacon node as a service so if the system restarts the process will automatically start back up again.

Loading config via .yaml file
In your Prysm working directory, create a .yaml file and open it in a text editor.

```bash
touch beacon_config.yaml
```

Add the following lines to the file before closing and saving

```bash
datadir: '/eth/prysm/data/beacon'
http-web3provider: 'https://goerli.infura.io/v3/abdecbed7270438cbfe704cdc26424cc'
accept-terms-of-use: true
log-file: '/eth/prysm/log/beacon.log'
genesis-state: '/eth/prysm/data/genesis/genesis.ssz'
```

Fix error:

```bash
curl https://raw.githubusercontent.com/eth2-clients/eth2-networks/master/shared/pyrmont/genesis.ssz --output genesis.ssz
```

### Create and Configure the Service

Create a systemd service config file to configure the service.

```bash
nano /etc/systemd/system/prysmbeacon.service
```

Paste the following into the file

```bash
[Unit]
Description=Prysm Eth2 Client Beacon Node
After=network.target

[Service]
ExecStart=/eth/prysm/prysm.sh beacon-chain --config-file=/eth/prysm/beacon_config.yaml --pyrmont
User=root
Type=simple
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload systemd to reflect the changes.

```bash
systemctl daemon-reload
```

Start the service and check to make sure it’s running correctly.

```bash
systemctl start prysmbeacon
systemctl status prysmbeacon
```

Enable the service to automatically start on reboot.

```bash
systemctl enable prysmbeacon
```

follow the progress or check for errors

```bash
journalctl -fu prysmbeacon.service
```

## Step 4: Complete the Pyrmont On-boarding Process

The steps to sign-up are:

- Get Göerli ETH
- Generate the validator keys. Each key is a validator account
- Fund the validator account(s) with 32 Göerli ETH per account
- Wait for your validator account(s) to become active

### Get Test ETH

https://faucet.goerli.mudit.blog/

### Generate Validator Keys

Go here to get the “Latest release” of the deposit command line interface app

```bash
curl -LO https://github.com/ethereum/eth2.0-deposit-cli/releases/download/v1.2.0/eth2deposit-cli-256ea21-linux-amd64.tar.gz

```

Unpack the tar archive

```bash
tar xvf eth2deposit-cli-256ea21-linux-amd64.tar.gz
cd eth2deposit-cli-ed5a6d3-linux-amd64
```

Clean up by removing the downloaded tar archive file.

```bash
rm -rf eth2deposit-cli-ed5a6d3-linux-amd64.tar.gz
```

Run the application to generate the validator keys.

```bash
./deposit new-mnemonic --num_validators 1 --mnemonic_language=english --chain pyrmont
```

It will ask you to create a wallet password. We will use this to load the validator keys into your client’s validator wallet. Back it up somewhere safe.

Backup seed phrase and validator keystore password

![Alt-Tex](/images/validator_generation.png)

The newly created validator keys and deposit data file are created at the specified location.

The deposit_data-[timestamp].json file contains the public keys for the validators and information about the deposit. This file will be used to complete the deposit process in the next step. Since we are on a server we don’t have a web browser so secure FTP (SFTP) the file over to a computer running MetaMask.

```bash
sftp -P 10001 root@eth-node-01.lcx.cloud.css-it.net:/eth/prysm/eth2deposit-cli-256ea21-linux-amd64/validator_keys/deposit_data-1631529067.json /Users/florolf/Desktop
```

The keystore-m...json files contain the encrypted signing key. There is one keystore-m per validator. These will be used to create the client validator wallet.

### Fund the Validator Keys

This step involves depositing the required amount of Göerli ETH to the Pyrmont testnet staking contract. This is done on the Eth2.0 Lauchpad website.

Go here: https://pyrmont.launchpad.ethereum.org/

You will be asked to upload the deposit_data-[timestamp].json file. This should have been copied over in the previous step. Browse or drag the file and click continue.

Connect your wallet. Choose MetaMask, log in, select the Göerli Test Network and click Continue.

A summary shows the number of validators and total amount of Göerli ETH required. Tick the boxes if you agree and click continue.

### Check the Status of Your Validators

Newly added validators can take a while (hours to days) to activate. You can check the status of your keys with these steps:

- Copy your Göerli Test Network wallet address
- Go here: https://pyrmont.beaconcha.in/
- Search your wallet address. Your keys will be shown.
  Click on a key to see the Estimated Activation information.

## Step 5: Create the Validator Wallet

```bash
./prysm.sh validator accounts import --keys-dir=/eth/prysm/eth2deposit-cli-256ea21-linux-amd64/validator_keys --pyrmont
```

Enter new wallet directory

```bash
/eth/prysm/data/validator
```

You will be asked to provide a new wallet password. Make sure you keep it safe! We will need this later when configuring the validator.

Backup wallet password

Next you will need to enter the password you used to create the validator keys on the Eth2 Launch Pad site. If you enter it correctly the accounts will be imported into the new wallet.
![Alt-Tex](/images/validator_wallet.png)

Confirm the validator accounts have been created.

```bash
./prysm.sh validator -- accounts list --pyrmont --wallet-dir /eth/prysm/data/validator --accept-terms-of-use
```

![Alt-Tex](/images/validator_wallet_confirm.png)

Create a file to store the wallet password so the validator can access the wallet without having to manually supply the password. The file will be named password.txt.

```bash
cd data/validator/
touch password.txt
nano password.txt

```

Add your new wallet password to the file. Save and exit.

## Step 6: Configure the Validator

Loading config via .yaml file

1. In your Prysm working directory, create a .yaml file and open it in a text editor.

```bash
touch validator_config.yaml
```

2. Add the following lines to the file before closing and saving

```bash
datadir: '/eth/prysm/data/validator'
wallet-dir: '/eth/prysm/data/validator'
wallet-password-file: '/eth/prysm/data/validator/password.txt'
graffiti: ’florolf'
accept-terms-of-use: true
log-file: '/eth/prysm/log/validator.log'

```

Create a systemd service file to store the service config.

```bash
nano /etc/systemd/system/prysm-validator.service
```

Paste the following into the file

```bash
[Unit]
Description=Prysm Eth2 Client Validator Node
After=network.target

[Service]
ExecStart=/eth/prysm/prysm.sh validator --config-file=/eth/prysm/validator_config.yaml --pyrmont
User=root
Type=simple
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload systemd to reflect the changes.

```bash
systemctl daemon-reload
```

Start the service and check to make sure it’s running correctly.

```bash
systemctl start prysm-validator
systemctl status prysm-validator
```

Enable the service to automatically start on reboot.

```bash
systemctl enable prysm-validator
```

follow the progress or check for errors

```bash
journalctl -fu prysm-validator.service
```

It can take hours or days for the beacon chain to sync with your Eth1 node. It can take hours or days to activate the validation accounts(s) once the beacon chain has actually synced. The output from the validator process indicates the status.

You can check the status of your validator(s) via beaconcha.in. Simply do a search for your validator public key(s) or search using your MetaMask wallet address. It may be a while before they appear on the site.

## Monitoring

## Step 7: Install Prometheus

Prometheus is an open-source systems monitoring and alerting toolkit. It runs as a service on your Ubuntu server and its job is to capture metrics. More information here.
We are going to use Prometheus to expose runtime data from the beacon-chain and validator as well as instance specific metrics.

Create Directories

```bash
mkdir /eth/prometheus
```

Download Prometheus software

```bash
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.22.0/prometheus-2.22.0.linux-amd64.tar.gz
```

Unpack the archive. It contains two binaries and some content files.

```bash
tar xvf prometheus-2.22.0.linux-amd64.tar.gz
cp -a /eth/prometheus/prometheus-2.22.0.linux-amd64/. /eth/prometheus/
```

Remove the downloaded archive.

```bash
rm -rf prometheus-2.22.0.linux-amd64.tar.gz
```

Edit the Configuration File

```bash
nano /eth/prometheus/prometheus.yml
```

Paste the following into the file taking care not to make any additional edits and exit and save the file.

```bash
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: 'validator'
    static_configs:
      - targets: ['localhost:8081']
  - job_name: 'beacon node'
    static_configs:
      - targets: ['localhost:8080']
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

The scrape_configs define the output target for the different job names. We have 3 job names: validator, beacon node, and node_exporter. The first two are obvious, the last one is for metrics related to the server instance itself (memory, CPU, disk, network etc.). We will install and configure node_exporter below.

Run service

```bash
./prometheus --config.file /eth/prometheus/prometheus.yml \ --storage.tsdb.path /eth/prometheus/data --web.console.templates=/eth/prometheus/consoles --web.console.libraries=/eth/prometheus/console_libraries
```

Set Prometheus to Auto-Start as a Service

```bash
nano /etc/systemd/system/prometheus.service
```

Paste the following into the file. Exit and save.

```bash
[Unit]
Description=Prometheus
After=network.target

[Service]
ExecStart=/eth/prometheus/prometheus --config.file /eth/prometheus/prometheus.yml --storage.tsdb.path /eth/prometheus/data --web.console.templates=/eth/prometheus/consoles --web.console.libraries=/eth/prometheus/console_libraries
User=root
Type=simple
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload systemd to reflect the changes.

```bash
systemctl daemon-reload
```

And then start the service with the following command and check the status to make sure it’s running correctly.

```bash
systemctl start prometheus
systemctl status prometheus
```

Lastly, enable Prometheus to start on boot.

```bash
systemctl enable prometheus
```

## Step 8: Install Node Exporter

Prometheus will provide metrics about the beacon chain and validators. If we want metrics about our Ubuntu instance, we’ll need an extension called Node_Exporter. You can find the latest stable version here if you want to specify a different version below. Rpi users remember to get the ARM binary.

```bash
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
```

Unpack the downloaded software.

```bash
tar xvf node_exporter-1.2.2.linux-amd64.tar.gz
```

Remove the downloaded archive.

```bash
cp -a /eth/nodeexporter/node_exporter-1.2.2.linux-amd64/. /eth/nodeexporter/
rm -rf node_exporter-1.2.2.linux-amd64 node_exporter-1.2.2.linux-amd64.tar.gz
```

Set Node Exporter to Auto-Start as a Service

```bash
nano /etc/systemd/system/node_exporter.service
```

Paste the following into the file. Exit and save.

```bash
[Unit]
Description=Node Exporter
After=network.target

[Service]
ExecStart=/eth/nodeexporter/node_exporter
User=root
Type=simple

[Install]
WantedBy=multi-user.target
```

Reload systemd to reflect the changes.

```bash
systemctl daemon-reload
systemctl start node_exporter
systemctl status node_exporter
systemctl enable node_exporter
```

![Alt-Tex](/images/node_exporter.png)

## Step 8: Install Grafana

While Prometheus is our data source, Grafana is going provide our reporting dashboard capability.

Download the Grafana GPG key with wget, then pipe the output to apt-key. This will add the key to your APT installation’s list of trusted keys.

```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

Add the Grafana repository to the APT source

```bash
add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
```

Refresh the apt cache.

```bash
apt update
```

Make sure Grafana is installed from the repository.

```bash
apt-cache policy grafana
```

Verify the version at the top matches the latest version shown here. Then proceed with the installation.

```bash
apt install grafana
```

Start the Grafana server and check the status to make sure it’s running correctly.

### Configure Grafana Login

go to

```bash
http://195.201.108.158:3000/
```

![Alt-Tex](/images/grafana_dashboard.png)

in a browser and the Grafana login screen should come up.

Configure the Grafana Data Source

Click on Add Data Source and then choose Prometheus. Enter http://localhost:9090 for the URL then click on Save and Test.

Now let’s import a dashboard. Move your mouse over the + icon on the left menu bar. A menu will pop-up - choose Import.
Paste the JSON from here (or here if you have more than 10 validators) and click Load then Import. You should be able to view the dashboard. At first you may not have sufficient data, but after the testnet starts and validators are activated for a while you will see some metrics and alerts.

```bash
systemctl start grafana-server
systemctl status grafana-server
systemctl enable grafana-server
```

Monitor mobile via beaconchain

login in https://pyrmont.beaconcha.in/user/settings#app and follow docs

```bash
nano /etc/systemd/system/beaconchain.service
systemctl start beaconchain.service
systemctl status beaconchain.service
systemctl enable beaconchain.service
```

realated to:
https://someresat.medium.com/guide-to-staking-on-ethereum-2-0-ubuntu-pyrmont-prysm-a10b5129c7e3

# Mainnet

comming soon
