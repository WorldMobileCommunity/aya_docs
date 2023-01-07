---
sidebar_position: 1
---

# Installation

### Validator Install Script

```bash title="earthnode_setup.sh"
#!/usr/bin/env bash

# This function displays usage instructions for the script
usage() {
  # Display Usage
  echo "Aya Testnet node installation script."
  echo
  echo "Syntax: scriptTemplate -m <nodeMoniker> -a <operatorAccountName>"
  echo "options:"
  echo "a     Set you operator account name (stored locally)."
  echo "m     Set the node moniker."
  echo
}

# This function displays a message to contact support and exits the script
contact_support() {
  echo "Please contact support. "
  exit 1
}

# Set the path to the aya home directory
aya_home=/opt/aya

# Set the path to the current script
path=$(realpath "${BASH_SOURCE:-$0}")

# Set the path to the logfile using the current timestamp
logfile=$(dirname "${path}")/installation_$(date +%s).log

# Set the path to the cosmovisor logfile
cosmovisor_logfile=${aya_home}/logs/cosmovisor.log

# Set the path to the en_registration.json file
en_registration_json=${aya_home}/registration.json

# Clear the contents of the logfile
true >"$logfile"

# Initialize empty variables for the node moniker and operator account name
moniker=''
account=''

# Parse the command-line options passed to the script
# Set the value of the 'moniker' variable with the 'm' flag
# Set the value of the 'account' variable with the 'a' flag
while getopts 'a:m:v' flag; do
  case "${flag}" in
  a) account="${OPTARG}" ;;
  m) moniker="${OPTARG}" ;;
  # If any other flag is passed, display the usage and exit
  *)
    print_usage
    exit 1
    ;;
  esac
done

# If the 'moniker' or 'account' variables are empty, display an error message and the usage before exiting the script
if [[ ! "$moniker" ]] || [[ ! "$account" ]]; then
  echo "Arguments -a and -m must be provided"
  usage
  exit 1
fi

# If the 'jq' package is not installed, install it
if ! dpkg -s jq >/dev/null 2>&1; then
  echo -e "-- Installing dependencies (jq package)\n"
  sudo apt-get -q install jq -y >/dev/null 2>&1
fi

# Check the checksum of the 'ayad' binary against the 'release_checksums' file
# If the checksums do not match, exit the script with an error message
grep "$(sha256sum ayad)" release_checksums 1>/dev/null
if [[ $? -ne 0 ]]; then
  echo "Incorrect checksum of ayad binary"
  exit 1
fi

# Check the checksum of the 'cosmovisor' binary against the 'release_checksums' file
# If the checksums do not match, exit the script with an error message
grep "$(sha256sum cosmovisor)" release_checksums 1>/dev/null
if [[ $? -ne 0 ]]; then
  echo "Incorrect checksum of cosmovisor binary"
  exit 2
fi

# Create the necessary directories and copy the 'ayad' and 'cosmovisor' binaries to their respective locations
sudo mkdir -p $aya_home
sudo chown "${USER}" $aya_home
mkdir -p "${aya_home}"/cosmovisor/genesis/bin
mkdir -p "${aya_home}"/backup
mkdir -p "${aya_home}"/logs
mkdir -p "${aya_home}"/config

# Copy ayad and cosmovisor binaries the the appropriate directories. Suppress shell output.
cp ayad "${aya_home}"/cosmovisor/genesis/bin/ayad >/dev/null 2>&1
cp cosmovisor "${aya_home}"/cosmovisor/cosmovisor >/dev/null 2>&1

# Initialize the node with the specified 'moniker'
# If this fails, display an error message and call the 'contact_support()' function to exit
if ! ./ayad init "${moniker}" --chain-id aya_preview_001 --home ${aya_home} >"$logfile" 2>&1; then
  echo "Failed to initialize the node, please contact support "
  contact_support
fi

# Copy the 'genesis.json' file to the 'config' directory
cp genesis.json "${aya_home}"/config/genesis.json

echo -e "-- Creating operator account: \n"

# Generate a new operator account and store the JSON output in the 'operator_json' variable
operator_json=$(./ayad keys add "${account}" --output json --home ${aya_home})

# Extract the address from the 'operator_json' variable and store it in the 'operator_address' variable
operator_address=$(echo "$operator_json" | jq '.address' | sed 's/\"//g')

# Display the mnemonic and address of the operator account
echo -e "\n-- [ONLY FOR YOUR EYES] Store this information safely, the mnemonic is the only way to recover your account. \n"
echo "$operator_json" | jq

# Set the log level to 'error' in the 'config.toml' file
sed -i "s/log_level = \"info\"/log_level = \"error\"/g" "${aya_home}"/config/config.toml

# Set the seed nodes in the 'config.toml' file
sed -i "s/seeds = \"\"/seeds = \"d52a534b483219e2cd7a354764c05bd84c6ff12b@aya-preview-seeds.zanazetu.com:16656\"/g" "${aya_home}"/config/config.toml

# Export some environment variables
export DAEMON_NAME=ayad
export DAEMON_HOME="${aya_home}"
export DAEMON_DATA_BACKUP_DIR="${DAEMON_HOME}"/backup
export DAEMON_RESTART_AFTER_UPGRADE=true
export DAEMON_ALLOW_DOWNLOAD_BINARIES=true

#Start cosmovisor. Apprend output to log file. Run in the background so script can continue.
"${aya_home}"/cosmovisor/cosmovisor run start --home ${aya_home} &>>"${cosmovisor_logfile}" &
#Verify cosmovisor process is running. If its not running display status in terminal and log file. Proceed to call contact support function.
if ! pgrep cosmovisor >/dev/null 2>&1; then
  echo "Cosmovisor not running." | tee -a "$logfile"
  contact_support
fi

echo -e "\n-- Copy the next json structure on the web form to lock your EN NFT and become a validator: \n"

# Get the address of the validator
validator_address=$(./ayad tendermint show-address --home ${aya_home})

# Use 'jq' to create a JSON object with the 'moniker', 'operator_address', and 'validator_address' fields
jq --arg key0 'moniker' \
  --arg value0 "$moniker" \
  --arg key1 'operator_address' \
  --arg value1 "$operator_address" \
  --arg key2 'validator_address' \
  --arg value2 "$validator_address" \
  '. | .[$key0]=$value0 | .[$key1]=$value1 | .[$key2]=$value2' \
  <<<'{}' | tee -a $en_registration_json

echo -e "\n-- Now we have to wait until your node is up to date... It will take a while!\n"

# Sleep for 30 seconds
sleep 30

# Set authorized to false
authorized=false

# While authorized is false, do the following:
while [ "$authorized" = "false" ]; do
  # If the balance of the operator address is 'uswmt', set authorized to true
  if [ "$(./ayad query bank balances "$operator_address" --output json 2>/dev/null | jq '.balances[].denom' | sed 's/\"//g')" = "uswmt" ]; then
    authorized=true
  # If the balance of the operator address is not 'uswmt', print a message and sleep for 60 seconds
  else
    echo "Still syncing... Height: $(./ayad status | jq '.SyncInfo.latest_block_height' | sed 's/"//g')"
    sleep 10
  fi
done

echo -e "\n-- All up to date! Becoming a validator now!\n"

# Create a validator tx using the 'ayad tx staking create-validator' command
if ! ./ayad tx staking create-validator \
  --amount=1uswmt \
  --pubkey="$(./ayad tendermint show-validator --home ${aya_home})" \
  --moniker="$moniker" \
  --chain-id="aya_preview_001" \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --from=$account \
  --home ${aya_home} \
  --output json \
  --yes ; then
  # If the 'ayad tx staking create-validator' command fails, print an error message and call the 'contact_support' function
  echo "Create validator transaction failed." | tee -a "$logfile"
  contact_support
fi

echo -e "\n-- All done on your side! Now WM will delegate your stake tokens!\n"
echo -e "\n-- Welcome to Aya sidechain :D \n\n"

echo -e "-- Configuring your node to start on server startup\n"

# Create symbolic links for the 'ayad' and 'cosmovisor' binaries
sudo ln -s $aya_home/cosmovisor/current/bin/ayad /usr/local/bin/ayad >/dev/null 2>&1
sudo ln -s $aya_home/cosmovisor/cosmovisor /usr/local/bin/cosmovisor >/dev/null 2>&1

# Create systemd service file that describes the background service running the 'cosmovisor' daemon.
sudo tee /etc/systemd/system/cosmovisor.service > /dev/null <<EOF
# Start the 'cosmovisor' daemon and append any output to the 'aya.log' file
# Create a Systemd service file for the 'cosmovisor' daemon
[Unit]
Description=Aya Node
After=network-online.target

[Service]
User=$USER
# Start the 'cosmovisor' daemon with the 'run start' command and write output to 'aya.log' file
ExecStart=$(which cosmovisor) run start --home "$aya_home" &>>"$aya_home"/logs/aya.log
# Restart the service if it fails
Restart=always
# Restart the service after 3 seconds if it fails
RestartSec=3
# Set the maximum number of file descriptors
LimitNOFILE=4096

# Set environment variables for data backups, automatic downloading of binaries, and automatic restarts after upgrades
Environment="/opt/aya"
Environment="DAEMON_NAME=ayad"
Environment="DAEMON_DATA_BACKUP_DIR=/opt/aya/backup"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_HOME=/opt/aya"
[Install]
# Start the service on system boot
WantedBy=multi-user.target
EOF

# Reload the Systemd daemon
sudo systemctl daemon-reload
# Enable the 'cosmovisor' service to start on system boot
sudo systemctl enable cosmovisor

```
