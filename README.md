# Autoclock Validator
* A fast and straightforward way to spin up a Solana validator running the Jito client. 
* The Autoclock Validator Ansible script has been designed and tested on c3.large [Latitude](https://www.latitude.sh/pricing) servers running Ubuntu thus far. Other OS and machine/disk configurations are untested yet, but feel free to fork or submit PRs to support additional infra.
* C3.large machines have 2 disks. One of these is mounted to / and the second one needs to be supplied in the defaults.

## Steps:

### If you don't already have a validator, start with the Solana docs:
1) Install Solana Tool Suite https://docs.solana.com/cli/install-solana-cli-tools
2) Create validator identity https://docs.solana.com/running-validator/validator-start#generate-identity
3) Create withdrawer account https://docs.solana.com/running-validator/validator-start#create-authorized-withdrawer-account
4) Create vote account https://docs.solana.com/running-validator/validator-start#create-authorized-withdrawer-account
5) 

<<7Layer question: the validator identity needs to be on the server right? we should have a step that mentions moving the validator identity keypair to the server when migrating?>>

### 1) SSH into your new server

### 2) Start a screen session
```
screen -S sol
```

### 3) Install Ansible
```
sudo apt-get update && sudo apt-get install ansible -y
```

### 4) Clone autoclock-validator repo and cd into folder
```
git clone https://github.com/overclock-validator/autoclock-validator.git
```
```
cd autoclock-validator
```

### 5) Edit the hosts.yaml file in the root location to point to your validator's IP address and the ssh parameters
* Example hosts.yaml: https://github.com/Overclock-Validator/autoclock-validator/blob/master/hosts.example.yaml

### 6) Edit and configure the common main.yaml file 
* https://github.com/overclock-validator/autoclock-validator/blob/master/roles/common/defaults/main.yaml
* Inside this file you will see (and may need to edit):
```
ledger_disk: "nvme1n1"
swap_mb: 100000
```
* ledger_disk needs to point to the nvme disk on the c3.large that is not currently mounted, which you can find using `lsblk`
* The Ansible script puts ledger on a separate disk and everything else (accounts, snapshots, OS) on the default disk (ledger and snapshot are both write intensive, so it's good to separate those to different disks).
* By default, swap_mb is set to 100gb, but for validators it's not that helpful outside of preventing a crash. If your machine is swapping however, there are other issues that need to be solved anyway.

### 7) Edit and configure the jito main.yaml file
* https://github.com/overclock-validator/autoclock-validator/blob/master/roles/jito/defaults/main.yaml
* Inside this file you will see (and may need to edit):
```
# 1. Supply a valid cluster
# testnet, mainnet
cluster: "mainnet"

# 2. Supply a valid jito block engine location
# mainnet: amsterdam, frankfurt, ny, tokyo 
# testnet: dallas, ny
location: "ny"

# Commission is in basis points (bps). 100 bps = 1%
commission-bps: 800
# Optional. This sends metricss to Solana's public influx and it is encouraged to set to true since it helps Solana Labs and others debug your validator as well as network issues.
metrics: true
org_name: "jito-foundation"
repo_name: "jito-solana"
repo_dir: "jito-solana"
repo_version: "v1.13.6-jito"
```
* The location as mentioned in the comment needs to be one of the 4 (for mainnet) or one of 2 (for testnet) - the validator parameters for block engine, relayer, etc are set based on the nearest location to your validator server.
* The repo_version needs to be modified to whichever tag you want the validator to run. Consult Jito discord (link below) for the latest version expected to be run.
* Other parameters can be left as is (most validators set commission to 800 basis points atm, but you can adjust that if you want to).

### 8) Edit and configure below command and run Ansible
* This step assumes that validator-keypair.json and vote-account-keypair.json have been generated using solana-keygen and that the vote-account has already been created. The Ansible playbook executes the vote-account command to see that vote-account-keypair.json actually exists and is associated with validator-keypair.json. It will fail before starting the validator if that is not the case.
* Make sure that the solana_version is up to date and that you specify the correct cluster
```
ansible-playbook setup.yaml -i hosts.yaml -e id_path=./keys/validator-keypair.json -e vote_path=./keys/vote-account-keypair.json -e region=ny -e cluster=testnet -e rpc_address=https://api.testnet.solana.com -e repo_version=v1.14.16-jito
```
* This command can take between 10-20 minutes based on the specs of the machine
* It takes long because it does everything necessary to start the validator (format disks, checkout the solana repo and build it, download the latest snapshot, etc.)

### 9) Once Ansible finishes, switch to the Solana user with:
```
sudo su - solana
```

### 10) Check the status

```
source ~/.profile
solana-validator --ledger /mnt/solana-ledger monitor
ledger monitor
Ledger location: /mnt/solana-ledger
⠉ Validator startup: SearchingForRpcService...
```

#### Initially the monitor should just show the below message which will last for a few minutes and is normal:

```
⠉ Validator startup: SearchingForRpcService...
```

#### After a while, the message at the terminal should change to something similar to this:

```
⠐ 00:08:26 | Processed Slot: 156831951 | Confirmed Slot: 156831951 | Finalized Slot: 156831917 | Full Snapshot Slot: 156813730 |
```

#### Check whether the RPC is caught up with the rest of the cluster with:

```
solana catchup --our-localhost
```

If you see the message above, then everything is working fine! Gratz. You have a new RPC server and you can visit the URL at http://xx.xx.xx.xx:8899/


??? ### 1) Install Ansible locally ????
```
https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html
```

# Helpful Links
* Solana Discord (use validator-support channel)
* Jito Discord
* Overclock Discord

# TODO
* support different disk configurations
* support skipping disk setup
* support flag to make starting the validator optional
* support Labs client as well, later Firedancer
