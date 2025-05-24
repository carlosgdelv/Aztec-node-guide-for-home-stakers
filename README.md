# AZTEC-NODE-GUIDE-FOR-DEGENS


___


# Geth-Prysm-node
Step by step guide for setting up a `docker-compose.yml` for running a `Sepolia` Ethereum full node using **Geth** as the `execution client` and **Prysm** as the `consensus client` on an Ubuntu-based system.

___




## Hardware Requirements (Recommendation)
<table>
  <tr>
    <th colspan="3"> OS: Ubuntu 20.04 or later</th>
  </tr>
  <tr>
    <td>CPU</td>
    <td>ram</td>
    <td>Disk</td>
  </tr>
  <tr>
    <td><code>6-8 cores</code></td>
    <td><code>16 GB DDR5</code></td>
    <td><code>1.5 TB SSD</code></td>
  </tr>
</table>


## My personal PC specifications for Running a Node


- **Processor (CPU)**  
  Intel Core i7 (14th Gen) with 20 cores (8 Performance + 12 Efficient), 3.4 GHz base and 5.6 GHz max turbo.

- **Memory (RAM)**  
  32GB DDR5 (2×16GB) at 6000 MHz with CL32 latency in dual-channel configuration.

- **Storage**  
  2TB SSD using PCIe 4.0 NVMe interface with Gen 4x4 for high-speed data transfer.

- **Power Supply (PSU)**  
  750W unit with 80 Plus Bronze certification for efficient energy usage.

- **Cooling**  
  240mm liquid cooler with dual fans and RGB lighting for effective thermal management.

- **Motherboard**  
  ATX board with B760 chipset, supports DDR5, PCIe 4.0, and includes built-in Wi-Fi.

---

## Step 1. Install Dependecies
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
Add Your User to the Docker Group
Adds your current Linux user to the docker group so you can run Docker commands without using sudo.
```bash
sudo usermod -aG docker $USER
```
> Note: You must log out and log back in (or reboot) for this change to take effect in new terminals.

Applies the new group membership immediately in the current terminal session without needing to log out.
```bash
newgrp docker
```


---

## Step 2. Create Directories
These commands create the necessary directory structure for Ethereum's execution (/execution) and consensus (/consensus) clients under /root/ethereum, ensuring the paths exist for storing their respective data.
```bash
sudo mkdir -p /root/ethereum/execution
sudo mkdir -p /root/ethereum/consensus
```

---

## Step 3. Generate the JWT secret:
Generates a 32-byte random JWT secret in hexadecimal format and saves it to a file used for secure communication between clients.

Generates a random 32-byte hexadecimal string using OpenSSL and saves it to the file /root/ethereum/jwt.hex with root permissions.
```bash
sudo bash -c "openssl rand -hex 32 > /root/ethereum/jwt.hex"
```
Displays the contents of the /root/ethereum/jwt.hex file with root permissions.
```bash
sudo cat /root/ethereum/jwt.hex
```

---

## Step 4. Configure `docker-compose.yml`
Changes the current working directory to the `ethereum` folder where you will place the Docker Compose configuration.
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
    image: ethereum/client-go:stable
    container_name: geth
    network_mode: host
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /home/<Linuxuser>/execution:/data
      - /home/<Linuxuser>/jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.api=eth,net,web3
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain
    container_name: prysm
    network_mode: host
    restart: unless-stopped
    volumes:
      - /home/<Linuxuser>/consensus:/data
      - /home/<Linuxuser>/jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
    command:
      - --sepolia
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=0.0.0.0
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --min-sync-peers=3
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

If a port is in use, edit the `docker-compose.yml` to map to a different host port (e.g., `8546:8545` to `8547:8545`).

___

## Step 6. Run Geth & Prysm Nodes

Start Geth & Prysm Nodes:
Starts the Geth and Prysm containers in detached mode (running in the background).
```bash
docker compose up -d
```

Node Logs
Continuously displays the real-time logs from both containers.
```bash
# All lines of logs:
docker compose logs -f
```
___

## Step 7. Checking If Nodes Are Synced
➡️**Execution Node (Geth)**
```
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```
*✅Response if fully synced:
```json
{"jsonrpc":"2.0","id":1,"result":false}
```
🚫Response if still syncing:
```json
{"jsonrpc":"2.0","id":1,"result":{"currentBlock":"0x1a2b3c","highestBlock":"0x1a2b4d","startingBlock":"0x0"}}
```
You'll see an object with `startingBlock`, `currentBlock`, and `highestBlock`, indicating the sync progress.

➡️**Beacon Node (Prysm)**
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

---


## Step 8. Getting the RPC Endpoints
### Execution Node (Geth)
* **Aztec Sequencer Execution RPC (Running by `docker-compose.yml`)**: `http://127.0.0.1:8545` or `http://localhost:8545`

### Beacon Node (Prysm)
* **Aztec Sequencer Consensus Beacon RPC (Running by `docker-compose.yml`)**: `http://127.0.0.1:3500` or `http://localhost:3500`

> [Aztec Sequencer Node Guide](https://github.com/0xmoei/aztec-network)

---


# Aztec-node
Step by step guide for setting up a `docker-compose.yml` for running a `Sepolia` Aztec full node and Validator registration on an Ubuntu-based system.

## Step 1. Create Directory
Creates the `nodeaztec/aztec` directory under your home folder, including any parent directories, to store Aztec’s binaries and data.
```bash
mkdir -p ~/nodeaztec/aztec
```
## Step 2. Add Aztec CLI to Your System PATH
This appends the Aztec binary path (~/.aztec/bin) to your PATH environment variable in the .bashrc file, so that your system can recognize the aztec command from any terminal.```bash
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
```
This reloads the .bashrc file to apply the updated PATH immediately without needing to restart the terminal.```bash
source ~/.bashrc
```

## Step 3. Install Aztec
Downloads and runs Aztec’s official installer script in an interactive Bash shell, installing the latest Aztec CLI tools.
```bash
bash -i <(curl -s https://install.aztec.network)
```
Appends the Aztec binary directory to your shell’s `PATH` by updating your `~/.bashrc`, so you can run `aztec` from any location.
```bash
echo 'export PATH=$PATH:/root/.aztec/bin' >> ~/.bashrc
```
Reloads your `~/.bashrc` file into the current shell session, applying the updated `PATH` immediately without logging out and back in.
```bash
source ~/.bashrc
```
~ Now check if Aztec successfully installed
Runs the Aztec CLI with no arguments to verify that the command is available and prints its usage/help text, confirming a successful installation.
```bash
aztec
```
~ Then update Aztec to Alpha Testnet
Switches or initializes your Aztec environment to the Alpha Testnet configuration, downloading any required network artifacts and setting your CLI to target that test network.
```bash
aztec-up alpha-testnet
```

## Step 4. Configure `docker-compose.yml`

Changes your working directory to the `nodeaztec` folder where your Aztec node’s Docker Compose configuration is located.
```bash
cd nodeaztec
```
 Opens the `docker-compose.yml` file in the Nano text editor so you can define and configure all your containerized services.
```bash
nano docker-compose.yml
```


Replace the following code into your `docker-compose.yml` file:
```yaml

services:
  node:
    image: aztecprotocol/aztec:0.85.0-alpha-testnet.5
    network_mode: host # Debe ir aquí, dentro del servicio
    environment:
      ETHEREUM_HOSTS: "http://localhost:8545"
      L1_CONSENSUS_HOST_URLS: "http://localhost:3500"
      DATA_DIRECTORY: /data
      VALIDATOR_PRIVATE_KEY: 
      P2P_IP: 
      LOG_LEVEL: debug
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet start --node --archiver --sequencer'
    ports:
      - 40400:40400/tcp
      - 40400:40400/udp
      - 8080:8080
```
Starts all services defined in your `docker-compose.yml` in the background (detached mode)
```bash
docker compose up -d
```
Streams real-time combined logs from all running Compose services, letting you watch your nodes’ output and troubleshoot as they run.
```bash
docker compose logs -f
```
Stop and Kill Node
```bash
docker compose down -v
```
___

## Update Sequencer Node
* 1- Stop Node:
```console
# CLI
docker stop $(docker ps -q --filter "ancestor=aztecprotocol/aztec") && docker rm $(docker ps -a -q --filter "ancestor=aztecprotocol/aztec")

screen -ls | grep -i aztec | awk '{print $1}' | xargs -I {} screen -X -S {} quit

# Docker
docker compose down
```

* 2- Update Node:
```bash
aztec-up alpha-testnet

docker compose pull
```

* 3- Delete old data:
This command forcefully deletes the entire Aztec alpha-testnet data directory and all its contents from your home folder to reset the node.
```bash
rm -rf ~/.aztec/alpha-testnet/data/
```

* 4- Re-run Node

Return to Step 4 to re-run your node



___


## Register Validator
Make sure your Sequencer node is fully synced, before you proceed with Validator registration
```bash
aztec add-l1-validator \
  --l1-rpc-urls RPC_URL \
  --private-key your-private-key \
  --attester your-validator-address \
  --proposer-eoa your-validator-address \
  --staking-asset-handler 0xF739D03e98e23A7B65940848aBA8921fF3bAc4b2 \
  --l1-chain-id 11155111
```
## Verify Node's Peer ID:
**Find your Node's Peer ID:**
```bash
sudo docker logs $(docker ps -q --filter ancestor=aztecprotocol/aztec:alpha-testnet | head -n 1) 2>&1 | grep -i "peerId" | grep -o '"peerId":"[^"]*"' | cut -d'"' -f4 | head -n 1
```
* This reveals your Node's Peer ID, Now search it on [Nethermind Explorer](https://aztec.nethermind.io/)
* Note: It might takes some hours for your node to show up in Nethermind Explorer after it fully synced.


___


## Sequencer/Validator Health Check
* Validator attestation stats:

https://t.me/aztec_seer_bot

![image](https://github.com/user-attachments/assets/04ca9f5d-ba72-43be-98ad-3601255000bf)

___

## - Getting Apprentice Role:
Head to Aztec Discord and go to `operator | start-here` channel
Run command `/operator help` there
Run this command:
```bash
curl -s -X POST -H 'Content-Type: application/json' \
-d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":67}' \
http://localhost:8080 | jq -r ".result.proven.number"
```
~ Change `http://localhost:8080` with your VPS IP:8080
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
___

## Monitor System
* Monitor your hardware usage:
```bash
htop
```

* Monitor your Disk usage:
```bash
df -h
```

* Monitor Geth Disk usage:
```bash
docker exec -it geth du -sh /data
```

* Monitor Prysm Disk usage:
```bash
docker exec -it prysm du -sh /data
```
* Monitor Aztec Disk usage:
```bash
docker exec -it nodeaztec-node-1 du -sh /data/*
```
___

# Simple Docker Guide

## Check running containers
Shows all currently running Docker containers.
```bash
docker ps
```

## See all containers (running or stopped)
Lists all containers, including those that are stopped.
```bash
docker ps -a
```

## Stop a running container
Stops the specified running container gracefully.
```bash
docker stop <container_id_or_name>
```

## Remove a container
Deletes the specified container permanently.
```bash
docker rm <container_id_or_name>
```

## Remove all stopped containers
Removes all stopped containers in one go to free up space.
```bash
docker container prune
```
## Uninstall Docker
Uninstalls Docker packages and deletes Docker data directories to fully clean your system.
```bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```









