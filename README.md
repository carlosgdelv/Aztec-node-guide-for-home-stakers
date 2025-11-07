# Aztec Node Guide for Home Stakers

___

## 🎯 Reasons to Run your Own Ethereum Node

To operate an Aztec node reliably, it is necessary to also run your own Ethereum execution and consensus clients. Aztec depends on the Ethereum base layer for data availability and settlement, which means it constantly queries and submits transactions to Ethereum. By managing your own Ethereum clients, you ensure low-latency, high-availability RPC access, reducing the risk of downtime, rate-limiting, or outages caused by third-party providers. Past issues with public RPCs have led to Aztec nodes failing to sync or sequence correctly, so running self-hosted clients is the most resilient and technically sound approach.

___

## 📚 Reference Documentation


Below you’ll find links to the official Aztec documentation, along with community-contributed resources designed to help you deploy, configure, and operate Aztec nodes:

- https://docs.aztec.network/the_aztec_network/guides/run_nodes/how_to_run_sequencer
- https://docs.aztec.network/the_aztec_network/guides/run_nodes/cli_reference
- https://github.com/0xmoei/geth-prysm-node
- https://github.com/0xmoei/aztec-network/blob/883a8ecc1772accef47c1a2f9d655d34889ad176/README.md#9-run-sequencer-node
- https://github.com/frianowzki/aztec-sequencer-node
- https://aztec.starfrich.me/
- https://dashtec.xyz/
- https://aztec.denodes.app/dashboard

___




##  🛡️ Suggested Hardware & Bandwidth Requirements
<table>
  <tr>
    <th colspan="4"> OS: Ubuntu 20.04 or later</th>
  </tr>
  <tr>
    <td>RAM</td>
    <td>CPU</td>
    <td>Disk</td>
    <td>Bandwidth</td>
  </tr>
  <tr>
    <td><code>16 GB DDR5</code></td>
    <td><code>6-8 cores</code></td>
    <td><code>2-3 TB SSD</code></td>
    <td><code>600 Mbps</code></td>
  </tr>
</table>


## 🖥️ Personal PC Specifications for Running a Node


- **Processor (CPU)**  
  Intel Core i7 (14th Gen) with 20 cores (8 Performance + 12 Efficient), 3.4 GHz base and 5.6 GHz max turbo.

- **Memory (RAM)**  
  32GB DDR5 (2×16GB) at 6000 MHz with CL32 latency in dual-channel configuration.

- **Storage**  
  2TB SSD using PCIe 4.0 NVMe interface with Gen 4x4 for high-speed data transfer.

- **Power Supply (PSU)**  
  750W unit with 80 Plus Bronze certification for efficient energy usage.

- **Cooling**  
  240mm liquid cooler with dual fans.

- **Motherboard**  
  ATX board with B760 chipset, supports DDR5 and PCIe 4.0.

___


## 🪟 WSL installation 

Install Windows Subsystem for Linux (WSL) on this machine.
```bash
wsl --install
```
Restart the computer so WSL installation and kernel updates take effect.
```bash
(Restart your PC)
```
List Linux distributions available to install from the Microsoft catalog.
```bash
wsl --list --online
```
Install Ubuntu 24.04 into WSL.
```bash
wsl --install -d Ubuntu-24.04
```
Open an interactive shell inside the Ubuntu 24.04 WSL distribution.
```bash
wsl -d Ubuntu-24.04
```
or
```bash
wsl
```
___


# ⟠  Eth-Prysm-node
Step by step guide for setting up a `docker-compose.yml` for running a `Sepolia` Ethereum full node using **Geth** as the `execution client` and **Prysm** as the `consensus client` on an Ubuntu-based system.

___

## Step 1. 🔧 Install Dependecies
**Packages:**
Refreshes the local package index and then upgrades all installed packages to their latest available versions without prompting for confirmation.
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
This command installs a wide range of essential development tools, system utilities, and libraries required to build, run, and manage blockchain nodes and Docker-based environments on a Linux system.
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev  -y
```
**Docker:**
The system is first updated and any previous Docker-related packages are removed to prevent conflicts. Then, necessary certificates and tools like `gnupg` are installed to securely add Docker’s official GPG key. A trusted keyring directory is created, and Docker’s repository is added to the system’s sources list. After updating the package list again, the official Docker Engine, CLI, and plugins are installed from Docker’s repository. Finally, Docker is tested with `hello-world`, enabled to start on boot, and restarted to ensure it runs properly.
```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```


---

## Step 2. 👤➕🐳 Add Your User to the Docker Group

It must show "carlos"
```bash
whoami
```  
It must show "1000"
```bash
id -u
```
It must show "1000"
```bash
id -g        
```
It should say: groupadd: the group "docker" already exists
```bash
sudo groupadd docker
```
The following command adds the user carlos to the docker group so they can use Docker without sudo
```bash
sudo usermod -aG docker carlos
```
Restart the session or the system
```bash
reboot
```


## Step 3. 📁 Create Directories
These commands create the necessary directory structure for Ethereum's execution and consensus clients under the user's home directory (`~/ethereum`):
```bash
mkdir -p ~/ethereum/execution ~/ethereum/consensus
```

---

## Step 4. 🔐 Generate the JWT secret:
Generates a 32-byte random JWT secret in hexadecimal format and saves it to a file used for secure communication between clients.

```bash
openssl rand -hex 32 > ~/ethereum/jwt.hex
```

Protect the JWT secret with strict permissions
```bash
chmod 600 /home/carlos/ethereum/jwt.hex
```

Prints the contents of the jwt.hex file to verify it was correctly generated.
```bash
cat ~/ethereum/jwt.hex
```

---

## Step 5. 🐳 Configure `docker-compose.yml`
Changes the current working directory to the `ethereum` folder where you will place the `docker-compose.yml ` configuration.
```bash
cd ~/ethereum
```
Opens a new or existing `docker-compose.yml` file in the Nano text editor to write or edit the service definitions.
```bash
nano docker-compose.yml
```
Replace the following code into your `docker-compose.yml` file:
```yaml
services:
  geth:
    image: ethereum/client-go:v1.16.4
    container_name: geth
    network_mode: host
    user: "1000:1000"
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /home/carlos/ethereum/execution:/data
      - /home/carlos/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --mainnet
      - --http
      - --http.api=eth,net,web3
      - --http.addr=127.0.0.1
      - --authrpc.addr=127.0.0.1
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --cache=8192
      - --maxpeers=50
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain:v6.1.2
    container_name: prysm
    network_mode: host
    user: "1000:1000"
    restart: unless-stopped
    volumes:
      - /home/carlos/ethereum/consensus:/data
      - /home/carlos/ethereum/jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
    command:
      - --mainnet
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=127.0.0.1
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=127.0.0.1
      - --grpc-gateway-port=3500
      - --min-sync-peers=3
      - --subscribe-all-data-subnets
      - --checkpoint-sync-url=https://mainnet.checkpoint.sigp.io
      - --genesis-beacon-api-url=https://mainnet.checkpoint.sigp.io
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```


___

## Step 6. ▶️ Run Geth & Prysm Nodes

▶️ Start Geth & Prysm Nodes:
Starts the Geth and Prysm containers in detached mode (running in the background).
```bash
docker compose up -d
```

🧾 Node Logs
Continuously displays the real-time logs from both containers.
```bash
docker compose logs -f
```
⛔ Stop node. 
Run `docker compose down` to stop and remove all running Aztec containers before updating.
```bash
docker compose down
```
## Step 7. 🔥 UFW


✅ Aplicar reglas UFW:

```bash
# 1. Allow SSH access (only if you use SSH, otherwise omit)
sudo ufw allow OpenSSH

# 2. Allow P2P traffic required for Geth P2P, Geth P2P, Prysm P2P, Prysm gossip/block sync

sudo ufw allow 30303/tcp      
sudo ufw allow 30303/udp      
sudo ufw allow 13000/tcp     
sudo ufw allow 12000/udp      

# 3. Default policy: deny incoming traffic
sudo ufw default deny incoming

# 4. Allow outgoing traffic
sudo ufw default allow outgoing

# 5. Enable the firewall
sudo ufw enable

```

Used Ports 
```bash
sudo ss -tulnp
```
UFW rules
```bash
sudo ufw status verbose
```

___

## Step 8. 🔄 Checking If Nodes are Synced
**Execution Node (Geth)**
```
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```
✅Response if fully synced:
```json
{"jsonrpc":"2.0","id":1,"result":false}
```
🚫Response if still syncing:
```json
{"jsonrpc":"2.0","id":1,"result":{"currentBlock":"0x1a2b3c","highestBlock":"0x1a2b4d","startingBlock":"0x0"}}
```
You'll see an object with `startingBlock`, `currentBlock`, and `highestBlock`, indicating the sync progress.

**Beacon Node (Prysm)**
```bash
curl http://localhost:3500/eth/v1/node/syncing
```
✅Response if fully synced:
```json
{"data":{"head_slot":"12345","sync_distance":"0","is_syncing":false}}
```
If `is_syncing` is `false` and `sync_distance` is `0`, the beacon node is fully synced.

🚫Response if still syncing:
```json
{"data":{"head_slot":"12345","sync_distance":"100","is_syncing":true}}
```
If `is_syncing` is `true`, the node is still syncing, and `sync_distance` indicates how many slots behind it is.

___


## Step 9. 🔎 Getting the RPC Endpoints
### Execution Node (Geth)
Aztec Sequencer Execution RPC (Running by `docker-compose.yml`)**: `http://127.0.0.1:8545` or `http://localhost:8545`

### Beacon Node (Prysm)
Aztec Sequencer Consensus Beacon RPC (Running by `docker-compose.yml`)**: `http://127.0.0.1:3500` or `http://localhost:3500`

---

# 🟣 Aztec-node (Testnet)
Step by step guide for setting up a `docker-compose.yml` for running a `Sepolia` Aztec full node and Validator registration on an Ubuntu-based system.

___
## Step 1. 👛 Create a Wallet and Fund It
To interact with the Aztec node on Sepolia, we will create a new MetaMask wallet (https://metamask.io/) in order to obtain its public and private key pair, and then fund it with Sepolia ETH using a faucet (https://sepolia-faucet.pk910.de/) so it can perform on-chain actions.

___
 
## Step 2. 📁 Create Directory
Creates the `nodeaztec/aztec` directory under your home folder, including any parent directories, to store Aztec’s binaries and data.
```bash
mkdir -p ~/nodeaztec/aztec
```
___

## Step 3. ⚙️ Install Aztec
Downloads and runs Aztec’s official installer script in an interactive Bash shell, installing the latest Aztec CLI tools.
```bash
bash -i <(curl -s https://install.aztec.network)
```

## Step 4. 🧭 Add Aztec CLI to your System PATH
This appends the Aztec binary path (`~/.aztec/bin`) to your PATH environment variable in the `.bashrc` file, so that your system can recognize the aztec command from any terminal.

```bash
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
```
This reloads the .bashrc file to apply the updated PATH immediately without needing to restart the terminal.

```bash
source ~/.bashrc
```

___



Now check if Aztec successfully installed
Runs the Aztec CLI with no arguments to verify that the command is available and prints its usage/help text, confirming a successful installation.
```bash
aztec
```
Then update Aztec to Alpha Testnet
Switches or initializes your Aztec environment to the Alpha Testnet configuration, downloading any required network artifacts and setting your CLI to target that test network.
```bash
aztec-up alpha-testnet
```
___


## Step 5. 🔥 Enable Firewall & Open Ports
Enables the firewall and opens required ports for SSH access and for the Aztec sequencer to communicate.
```bash
# Firewall
sudo ufw allow 22
sudo ufw allow ssh
sudo ufw enable

# Sequencer
sudo ufw allow 40400
sudo ufw allow 8080
  
```

## Step 6. 🌐 Check Your Public and Local IPs

Use these commands to verify your public IP (used for external communication) and your local/internal IPs (used within your network or Docker).

```bash
# Shows your public IPv4 address
curl ipv4.icanhazip.com
# Lists all local IP addresses assigned to your machine; look for the local one (e.g., 192.168.x.x)
hostname -I
```
___


## Step 7. 🔀 Port Forwarding

Many internet providers use Carrier-Grade NAT (CG-NAT) to conserve IPv4 addresses. Under CG-NAT, multiple customers share a single public IP address, and your router is assigned a private IP by your ISP, not a true public one. As a result, you cannot receive unsolicited external connections or open ports properly, because incoming traffic cannot be uniquely routed to your home network.

To enable true port forwarding and make your nodes publicly accessible, you must contact your ISP and request a dedicated public IP address (static public IP). This change allows your router to be directly reachable from the internet and makes port forwarding possible.

When running multiple nodes on different PCs within your home network, opening ports on each PC’s firewall (ufw allow) only allows local access. To make the nodes accessible from outside your network, you must configure port forwarding on your router.

Where to do port forwarding:
* Log in to your router’s admin panel (usually at 192.168.1.1 or 192.168.0.1 via a web browser).
* Find the Port Forwarding, Virtual Server, or NAT section.


What to do:
Forward specific external ports to the internal IP addresses and ports of each PC running a node.

Example:
* Forward external port 40400 → internal IP 192.168.1.100, port 40400 (PC1)
* Forward external port 40401 → internal IP 192.168.1.101, port 40400 (PC2)

Tip:
To ensure your port forwarding rules remain valid, you need to assign static IP addresses or configure DHCP reservations for each PC running a node. This prevents your internal IP addresses (e.g., `192.168.1.101`) from changing over time due to automatic IP assignment by your router (DHCP).

```bash
192.168.1.1  
User: user
Password: xxxxxxxx
https://www.yougetsignal.com/tools/open-ports/ 
```

To configure a DHCP reservation, you’ll need the MAC address of each PC, which uniquely identifies its network interface and can be obtained by running the `ip a` command in the terminal.

MAC address
```bash
ip a
```

If you have correctly configured port forwarding on your router, verify the accessibility of your nodes and services using the following commands:


Lists all processes on the local machine that are currently using TCP or UDP port 40400 (internal port).
```bash
sudo lsof -i :40400
```
Shows which process is bound to internal port 8080 and actively listening for incoming connections, including PID and executable name.
```bash
sudo ss -tulnp | grep 8080
```
Attempts to establish a TCP connection to internal port 40404 on localhost (loopback), to check if a service is listening.
```bash
nc -vz localhost 40404
```
Tries to establish a TCP connection to external (public) IP 79.116.75.145 on external port 8080, to verify port forwarding or public availability.
```bash
nc -vz yourpublicIP 8080
```
Attempts to reach external IP 79.116.75.145 on external port 40400, verifying whether this port is exposed through the router via port forwarding.
```bash
nc -vz yourpublicIP 40400
```
Attempts to connect to another device within the local network (internal IP 192.168.1.135) on its internal port 40400, to test LAN-level connectivity.
```bash
nc -vz yourinternalIP 40400
```
___


## Step 8. 🐳 Configure `docker-compose.yml`

Changes your working directory to the `nodeaztec` folder where your Aztec node’s Docker Compose configuration is located.
```bash
cd ~/nodeaztec
```
 Opens the `docker-compose.yml` file in the Nano text editor so you can define and configure all your containerized services.
```bash
nano docker-compose.yml
```


Replace the following code into your `docker-compose.yml` file:
```yaml

services:
  node:
    container_name: aztec-sequencer
    image: aztecprotocol/aztec:alpha-testnet
    network_mode: host
    restart: unless-stopped
    environment:
      ETHEREUM_HOSTS: "http://localhost:8545"
      L1_CONSENSUS_HOST_URLS: "http://localhost:3500"
      DATA_DIRECTORY: /data
      VALIDATOR_PRIVATE_KEYS: "0xPrivateKey1,0xPrivateKey2,0xPrivateKey3"
      SEQ_PUBLISHER_PRIVATE_KEY: 0xPrivateKey1
      COINBASE: 0xPubliceKey
      GOVERNANCE_PAYLOAD: ${GOVERNANCE_PAYLOAD}
      LOG_LEVEL: debug
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet --node --archiver --sequencer --p2p-enabled true --p2p.listenAddress 0.0.0.0 --p2p.p2pIp IP --p2p.p2pPort 40400 --p2p.queryForIp false --port 8080'
    volumes:
      - /root/.aztec/alpha-testnet/data/:/data

```
▶️ Starts all services defined in your `docker-compose.yml` in the background (detached mode)
```bash
docker compose up -d
```
🧾 Streams real-time combined logs from all running Compose services, letting you watch your nodes’ output and troubleshoot as they run.
```bash
docker compose logs -f
```
⛔ Run `docker compose down` to stop and remove all running Aztec containers before updating.
```bash
docker compose down
```

___

# 🧰 Aztec Node Setup — Testnet 2.0.2

## Step 1. 🔐 Create folder structure with secure permissions
```bash
mkdir -m 700 -p ~/aztec-sequencer/keys ~/aztec-sequencer/data
```

## Step 2. 📄 Create the .env file
```bash
cd ~/aztec-sequencer
touch .env
```
```bash
nano .env
```

And in `.env`, something like this (adjust if you already have it):

```bash
DATA_DIRECTORY=./data
KEY_STORE_DIRECTORY=./keys
LOG_LEVEL=info
ETHEREUM_HOSTS=[your L1 execution endpoint, or a comma separated list if you have multiple]
L1_CONSENSUS_HOST_URLS=[your L1 consensus endpoint, or a comma separated list if you have multiple]
P2P_IP=[your external IP address]
P2P_PORT=40400
AZTEC_PORT=8080
AZTEC_ADMIN_PORT=8880
```

This command displays the contents of the `.env` file in the terminal.
```bash
cat .env
```

## Step 3. 🏷️ Crete The Aztec Address

Aztec CLI installed:
```bash
bash -i <(curl -s https://install.aztec.network)
```
Run this command in your terminal to temporarily add the directory to the PATH for this current session:
```bash
export PATH="$HOME/.aztec/bin:$PATH"
```
The testnet version installed:
```bash
aztec-up -v latest
```
Set the required environment variables:
```bash
export NODE_URL=https://aztec-testnet-fullnode.zkv.xyz
export SPONSORED_FPC_ADDRESS=0x299f255076aa461e4e94a843f0275303470a6b8ebe7cb44a471c66711151e529
```
Unlike sandbox, testnet has no pre-deployed accounts. You need to create your own:
```bash
aztec-wallet create-account \
    --register-only \
    --node-url $NODE_URL \
    --alias my-wallet
```
# Step 4. 🔐 Keystore Encryption (Attester)

1️⃣ Create a file with your private key (without 0x)
```bash
printf "aabb...887799" > /tmp/privatekey.txt
chmod 600 /tmp/privatekey.txt
```
2️⃣ Crear un archivo con la contraseña de cifrado
Create a file with the encryption password
Disable shell history (prevents the commands from being saved in `~/.bash_history`)
```bash
set +o history
```
3️⃣ Generate and display a 10-word passphrase ONE TIME ONLY (it is not saved to a variable or file):
```bash
grep -E '^[a-z]{5,}$' /usr/share/dict/words | shuf -n 10 | paste -sd ' ' -
```
## 4️⃣ WRITE THE PASSPHRASE ON PAPER
Confirm before continuing:

```bash
read -s -p "Apunta la passphrase en papel y pulsa ENTER para continuar..." ; echo
```

5️⃣ Clear the screen and scrollback (works in most modern terminals)
```bash
printf '\033c'
printf '\e[3J' 
clear
```
6️⃣ Restore shell history:
```bash
set -o history
```
7️⃣ Save the passphrase to a secure file
```bash
read -s -p "Introduce ahora la passphrase que escribiste en papel: " PASSWORD
echo
printf "%s" "$PASSWORD" > ~/aztec-sequencer/password.txt
chmod 600 ~/aztec-sequencer/password.txt
unset PASSWORD
```
8️⃣ Verify there is no trailing newline
```bash
hexdump -C ~/aztec-sequencer/password.txt | tail -n1
```

9️⃣ Check file permissions (DOES NOT display the passphrase)
```bash
ls -l ~/aztec/password.txt
wc -c ~/aztec/password.txt
```

🔟  Import the private key as an encrypted keystore
```bash
geth account import --keystore ~/aztec-sequencer/keys --password ~/aztec-sequencer/password.txt /tmp/privatekey.txt
```

⓫  Remove the temporary private key
```bash
shred -u /tmp/privatekey.txt
```
🔍 How to know exactly which file was created (and its path)

Directly from geth with the account list
```bash
geth account list --keystore ~/aztec-sequencer/keys
```
You will see something like:
```bash
Account #0: {0xabcdef1234567890} /home/usuario/aztec-sequencer/keys/UTC--2025-10-22T17-41-12.123Z--0xabcdef1234567890.json
```
That second value is exacly the `path` you must use in your `validators.json`.

## 📄 Step 5. — Configure your validators.json

Edit your JSON to point to that file and to the password. If only the attester is encrypted, and the other roles (`coinbase`,`publisher` ) are plaintext, it would look like this: Edita tu JSON para que apunte a ese archivo y a la contraseña.

```bash
{
  "schemaVersion": 1,
  "validators": [
    {
      "attester": {
        "path": "/home/usuario/aztec-sequencer/keys/UTC--2025-10-16T22-40-30.000Z--0xabcdef1234567890.json",
        "password_file": "/home/usuario/aztec-sequencer/password.txt"
      },
      "publisher": "0x1234567890abcdef1234567890abcdef12345678",
      "coinbase": "0x9876543210abcdef9876543210abcdef98765432",
      "feeRecipient": "0xabcdef1234567890abcdef1234567890abcdef12"
    }
  ]
}

```

🔐 NOTE:
Use "password_file" (not "password") if the software allows it — this way you don't leave the password written in the JSON in plain text. If it only accepts "password", you can leave it in clear text, but it is less secure.


## 🛡️ Step 6. Security and verification

Apply strict permissions:
```bash
chmod 700 ~/aztec-sequencer/keys
chmod 600 ~/aztec-sequencer/keys/*
chmod 600 ~/aztec-sequencer/password.txt
```

## Step 7. 🧪 Verifications

✅ Verify that the file exists

```bash
ls ~/aztec-sequencer/keys/UTC--*
```

✅ Verify that the keystore was imported:
```bash
geth account list --keystore ~/aztec-sequencer/keys
```

✅ Verify that the validators.json file is well-formed:
```bash
jq . ~/aztec-sequencer/validators.json
```

✅ Verify that you can read the address from the JSON
```bash
jq -r .address ~/aztec-sequencer/keys/UTC--*.json
```

✅ Verify the KDF encryption parameters:
```bash
jq .crypto.kdfparams ~/aztec-sequencer/keys/UTC--*.json
```

✅  Check the permissions of the password file
```bash
ls -l ~/aztec-sequencer/password.txt
```

✅ Check whether the password.txt file contains a newline (it should not)
```bash
hexdump -C ~/aztec-sequencer/password.txt | tail -n1
```
If it ends with 0a => newline, rewrite with printf

Recommended values:
```bash
"n" ≥ 262144 (cuanto más alto, más lento el brute force)
"r" ≥ 8
"p" ≥ 1
```

## Step 8. 🔧  Directory and user permissions

✅ Verify:
```bash
ls -l ~/aztec-sequencer
ls -l ~/aztec-sequencer/keys
id $USER

```
✅ That is correct if carlos has UID 1000. 

To confirm:
```bash
id carlos
```
🛠️  If for some reason the permissions are not correct, fix it:
```bash
sudo chown -R 1000:1000 ~/aztec-sequencer
```

## 🐳 9. Configure docker-compose.yml
Edit:
```bash
nano docker-compose.yml
```
Replace the following code into your `docker-compose.yml` file:
```yaml
services:
  aztec-sequencer:
    image: "aztecprotocol/aztec:2.0.2"
    container_name: "aztec-sequencer"
    network_mode: host
    user: "1000:1000"
    ports:
      - ${AZTEC_PORT}:${AZTEC_PORT}
      - ${AZTEC_ADMIN_PORT}:${AZTEC_ADMIN_PORT}
      - ${P2P_PORT}:${P2P_PORT}
      - ${P2P_PORT}:${P2P_PORT}/udp
    volumes:
      - ${DATA_DIRECTORY}:/var/lib/data
      - ${KEY_STORE_DIRECTORY}:/var/lib/keystore
    environment:
      KEY_STORE_DIRECTORY: /var/lib/keystore
      DATA_DIRECTORY: /var/lib/data
      LOG_LEVEL: ${LOG_LEVEL}
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      L1_CONSENSUS_HOST_URLS: ${L1_CONSENSUS_HOST_URLS}
      P2P_IP: ${P2P_IP}
      P2P_PORT: ${P2P_PORT}
      AZTEC_PORT: ${AZTEC_PORT}
      AZTEC_ADMIN_PORT: ${AZTEC_ADMIN_PORT}
    entrypoint: >-
      node
      --no-warnings
      /usr/src/yarn-project/aztec/dest/bin/index.js
      start
      --node
      --archiver
      --sequencer
      --network testnet
    restart: always
```
## 📂 10. Secure permissions of the mounted volumes

✅ Verify that the local directories (./data and ./keys) have permissions for UID 1000:
```bash
sudo chown -R 1000:1000 ~/aztec-sequencer/data
sudo chown -R 1000:1000 ~/aztec-sequencer/keys
```

## 🚀 11. Start & update the node

✅ Validate before bringing it up:
```bash
geth account list --keystore ~/aztec-sequencer/keys
```
▶️ Start the services:
```bash
docker compose up -d
```
🧾 View logs in real time:

```bash
docker compose logs -f
```
⛔ Stop the node:
```bash
docker compose down
```

🔄 Update Node:

Run `aztec-up alpha-testnet` to fetch the latest configuration files for the Alpha Testnet environment.
```bash
aztec-up alpha-testnet
```
Run `docker compose pull` to download the most recent Docker images defined in your Compose setup.
```bash
docker compose pull
```
Delete old data:
This command forcefully deletes the entire Aztec alpha-testnet data directory and all its contents from your home folder to reset the node.
```bash
rm -rf ~/.aztec/alpha-testnet/data/
```
Re-run Node

___

## 🧰 Tooling Installation

Install Foundry (a smart contract development toolkit)

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
```
Clone the Aztec repository
```bash
git clone https://github.com/AztecProtocol/aztec-packages.git
cd aztec-packages
```
Install Yarn (a JavaScript package manager)

```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update
sudo apt install yarn
yarn -v
```

___

## 🆔 Verify Node's Peer ID
**Find your Node's Peer ID:**
```bash
sudo docker logs $(docker ps -q --filter ancestor=aztecprotocol/aztec:alpha-testnet | head -n 1) 2>&1 | grep -i "peerId" | grep -o '"peerId":"[^"]*"' | cut -d'"' -f4 | head -n 1
```
* This reveals your Node's Peer ID, Now search it on [Nethermind Explorer](https://aztec.nethermind.io/)
* Note: It might takes some hours for your node to show up in Nethermind Explorer after it fully synced.
* Note: If you get no output, replace `alpha-testnet` with the specific version tag (e.g. `0.87.8`) to retrieve the peer ID.

___

## 🎓 Getting Apprentice Role
Head to Aztec Discord and go to `operator | start-here` channel
Run command `/operator help` there
Run this command:
```bash
curl -s -X POST -H 'Content-Type: application/json' \
-d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":67}' \
http://localhost:8080 | jq -r ".result.proven.number"
```
Change `http://localhost:8080` with your VPS IP:8080
You will get a BLOCK_NUMBER like `21000` for example, save it.

Now run this:

```bash
curl -s -X POST -H 'Content-Type: application/json' \
-d '{"jsonrpc":"2.0","method":"node_getArchiveSiblingPath","params":["BLOCK_NUMBER","BLOCK_NUMBER"],"id":67}' \
http://localhost:8080 | jq -r ".result"
```
~ Change 2X BLOCK_NUMBER with the `BLOCK_NUMBER` you get recently.

Copy the PROOF and save it.
___

Now head back to `operator | start-here` and run `/operator start` command.

~ Change `address` with your Sequencer Node EVM address.

~ Change `block-number` with BLOCK_NUMBER you got above.

~ Change `proof` with PROOF you got above.
Congratulations, now you have Apprentice role!

## Hit enter and wait for it.

If it successfully registered you can check it from operator | start-here and use the command /operator my-stats and enter your validator address.
NOTE: Currently there is a daily registration quota each day, if you missed it now you can try tomorrow.

___

## 🔍 Monitor System

Monitor your hardware usage:
```bash
htop
```

Monitor your Disk usage:
```bash
df -h
```

Monitor Geth Disk usage:
```bash
docker exec -it geth du -sh /data
```

Monitor Prysm Disk usage:
```bash
docker exec -it prysm du -sh /data
```
Monitor Aztec Disk usage:
```bash
docker exec -it nodeaztec-node-1 du -sh /data/*
```
___

## 🐳 Simple Docker Guide

Shows all currently running Docker containers.
```bash
docker ps
```

Lists all containers, including those that are stopped.
```bash
docker ps -a
```
It shows real-time metrics of running containers, including CPU, memory, network, and disk usage.
```bash
docker stats 
```

Stops the specified running container gracefully.
```bash
docker stop <container_id_or_name>
```

Deletes the specified container permanently.
```bash
docker rm <container_id_or_name>
```

Removes all stopped containers in one go to free up space.
```bash
docker container prune
```

Uninstalls Docker packages and deletes Docker data directories to fully clean your system.
```bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

___

## 🔧 Extending Disk Space on a Linux VM (LVM + SSD)

1. Update system and install LVM tools
Install the LVM2 utilities required to manage logical volumes.
```bash
sudo apt update
sudo apt install lvm2 -y
```

2. Wipe all data and prepare the new disk
Remove all existing filesystem/partition data and initialize the disk as a physical volume.
```bash
sudo wipefs --all /dev/nvme1n1
sudo sgdisk --zap-all /dev/nvme1n1
sudo pvcreate /dev/nvme1n1
```

3. Add the new disk to the existing volume group
Extend your volume group (ubuntu-vg) to include the new disk.
```bash
sudo vgextend ubuntu-vg /dev/nvme1n1
```

4. Confirm volume group changes
Check that the volume group now contains the new physical volume.
```bash
sudo vgs
```

5. Extend the logical volume to use all free space
Grow the logical volume (ubuntu-lv) with all available space from the volume group.
```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```
6. Resize the filesystem
Expand the filesystem to match the new size of the logical volume.
```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```
7. Verify the changes
Check the updated disk space and filesystem type.
```bash
df -h /
df -T /
```
8. Optional: View detailed LV info
Display full details of the extended logical volume.
```bash
sudo lvdisplay /dev/ubuntu-vg/ubuntu-lv
```
