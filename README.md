<div align="center">

🖥️

# Linux Operating System & Node Infrastructure Handbook

**Guía práctica de administración de sistemas Linux y despliegue de nodos blockchain (Ethereum L1 + Aztec L2)**

![Bash](https://img.shields.io/badge/Bash-4EAA25?style=flat&logo=gnubash&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=flat&logo=ubuntu&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Ethereum](https://img.shields.io/badge/Ethereum-3C3C3D?style=flat&logo=ethereum&logoColor=white)

![License](https://img.shields.io/badge/license-MIT-green)
![Maintained](https://img.shields.io/badge/maintained-yes-brightgreen)
![PRs Welcome](https://img.shields.io/badge/PRs-welcome-blue)

</div>

---

## 📖 Acerca de

Guía completa para desplegar un nodo Ethereum L1 (Geth + Prysm) y un nodo Aztec L2 — tanto full node como secuenciador/validador — en una máquina propia o VPS, pensada para home stakers que prefieren no depender de RPCs públicos de terceros.

> Esta versión corrige varios puntos de la guía original que estaban desactualizados o eran inseguros: la imagen de Prysm apuntaba a una organización que Offchain Labs está deprecando, el método de cifrado de la clave del atestador omitía por completo la clave BLS (sin ella, el validador no puede funcionar), y faltaban advertencias de seguridad alrededor de claves privadas en texto plano. Todo lo de abajo está verificado contra la documentación oficial de Aztec y Prysm a fecha de esta guía — y aun así, en proyectos que evolucionan tan rápido como este, comprueba siempre la versión más reciente antes de desplegar en producción.

## 📑 Tabla de contenidos

- [🎯 Reasons to Run your Own Ethereum Node](#-reasons-to-run-your-own-ethereum-node)
- [📚 Reference Documentation](#-reference-documentation)
- [🛡️ Hardware & Bandwidth Requirements](#️-suggested-hardware--bandwidth-requirements)
- [⚠️ Security Checklist Before You Start](#️-security-checklist-before-you-start)
- [⟠ Part 1 — Ethereum L1 Node (Geth + Prysm)](#-part-1--ethereum-l1-node-geth--prysm)
- [🟣 Part 2 — Aztec L2 Node](#-part-2--aztec-l2-node)
- [🧰 Optional: Developer Tooling](#-optional-developer-tooling)
- [🆔 Verify Node's Peer ID](#-verify-nodes-peer-id)
- [🎓 Getting Apprentice Role](#-getting-apprentice-role)
- [🔥 Combined Firewall Reference](#-combined-firewall-reference)
- [🗂️ .gitignore](#️-gitignore)

---

## 🎯 Reasons to Run your Own Ethereum Node

To operate an Aztec node reliably, it is necessary to also run your own Ethereum execution and consensus clients. Aztec depends on the Ethereum base layer for data availability and settlement, which means it constantly queries and submits transactions to Ethereum. By managing your own Ethereum clients, you ensure low-latency, high-availability RPC access, reducing the risk of downtime, rate-limiting, or outages caused by third-party providers. Past issues with public RPCs have led to Aztec nodes failing to sync or sequence correctly, so running self-hosted clients is the most resilient and technically sound approach.

## 📚 Reference Documentation

Official Aztec documentation, plus community-contributed resources for deploying and operating Aztec nodes:

- https://docs.aztec.network/the_aztec_network/guides/run_nodes/how_to_run_sequencer
- https://docs.aztec.network/the_aztec_network/guides/run_nodes/cli_reference
- https://docs.aztec.network/operate/operators/keystore/creating_keystores
- https://docs.aztec.network/operate/operators/keystore/storage-methods
- https://github.com/0xmoei/geth-prysm-node
- https://github.com/0xmoei/aztec-network
- https://github.com/frianowzki/aztec-sequencer-node
- https://aztec.starfrich.me/
- https://dashtec.xyz/
- https://aztec.denodes.app/dashboard

## 🛡️ Suggested Hardware & Bandwidth Requirements

<table>
  <tr>
    <th colspan="4">OS: Ubuntu 20.04 or later</th>
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

> If you enable Prysm's `--supernode` mode (full blob/data-column custody — see Part 1), budget significantly more RAM and bandwidth than the table above. It's not needed for normal solo staking; see the note in Part 1, Step 5.

### 🖥️ Personal PC Specifications for Running a Node

- **Processor (CPU):** Intel Core i7 (14th Gen), 20 cores (8P+12E), 3.4 GHz base / 5.6 GHz max turbo.
- **Memory (RAM):** 32GB DDR5 (2×16GB) at 6000 MHz, CL32, dual-channel.
- **Storage:** 2TB NVMe SSD, PCIe 4.0 Gen 4x4.
- **Power Supply (PSU):** 750W, 80 Plus Bronze.
- **Cooling:** 240mm AIO liquid cooler, dual fans.
- **Motherboard:** ATX, B760 chipset, DDR5 + PCIe 4.0 support.

## ⚠️ Security Checklist Before You Start

This guide handles real private keys (JWT secrets, validator attester/BLS keys, keystore passwords). Before you touch a terminal:

- Never commit `jwt.hex`, `password.txt`, `validators.json`, `keys/`, or `.env` files to git — see the [.gitignore](#️-gitignore) section at the end and add it to your repo **before** your first commit, not after.
- The `AZTEC_ADMIN_PORT` (8880) is an unauthenticated administrative API. It must never be reachable from outside the host — don't add a UFW rule for it, don't forward it on your router. See [Combined Firewall Reference](#-combined-firewall-reference).
- If you're following the "quick test" sequencer path with a plaintext private key in `.env`, treat that key as already compromised: use a fresh wallet, fund it with only what you're willing to lose, and never reuse it elsewhere.
- For anything beyond testnet experimentation, use the CLI-generated keystore (Part 2B) and keep strict file permissions (`chmod 600`) on every file that touches a private key.

---

# ⟠ Part 1 — Ethereum L1 Node (Geth + Prysm)

Step by step guide for setting up a `docker-compose.yml` to run a Mainnet Ethereum full node using **Geth** as the execution client and **Prysm** as the consensus client on Ubuntu.

## Step 1. 🔧 Install Dependencies

Refresh the package index and upgrade all installed packages:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

Install build tools and system utilities needed for blockchain nodes and Docker:

```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip -y
```

**Docker:**

```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove -y "$pkg"; done

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```

> Añadí `-y` al loop de `apt-get remove`: sin él, cada paquete pide confirmación interactiva y el script se queda colgado esperando input en una sesión no interactiva (ej. ejecutándolo vía SSH con `bash script.sh`).

## Step 2. 👤➕🐳 Add Your User to the Docker Group

```bash
whoami        # should print your username
id -u         # should print 1000 for the first non-root user
id -g         # should print 1000
sudo groupadd docker || true   # "already exists" is fine
sudo usermod -aG docker "$USER"
```

Apply the new group without a full reboot:

```bash
newgrp docker
```

(A reboot also works, but `newgrp docker` applies the group change to your current shell immediately — useful if you're running this remotely over SSH and don't want to risk a hung reconnect.)

> Generalicé `carlos` → `$USER` en todo el documento. Si vas a publicar esta guía para que otros la sigan, un username hardcodeado obliga a cada lector a editar cada comando a mano — y es fácil que se les pase uno.

## Step 3. 📁 Create Directories

```bash
mkdir -p ~/ethereum-mainnet/execution ~/ethereum-mainnet/consensus
```

## Step 4. 🔐 Generate the JWT Secret

```bash
openssl rand -hex 32 > ~/ethereum-mainnet/jwt.hex
chmod 600 ~/ethereum-mainnet/jwt.hex
cat ~/ethereum-mainnet/jwt.hex
```

## Step 5. 🐳 Configure `docker-compose.yml`

```bash
cd ~/ethereum-mainnet
nano docker-compose.yml
```

```yaml
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./execution:/data
      - ./jwt.hex:/data/jwt.hex
    command:
      - --mainnet
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
    image: gcr.io/offchainlabs/prysm/beacon-chain:stable
    container_name: prysm
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./consensus:/data
      - ./jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    command:
      - --mainnet
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
      - --checkpoint-sync-url=https://mainnet.checkpoint.sigp.io
      - --genesis-beacon-api-url=https://mainnet.checkpoint.sigp.io
      - --subscribe-all-subnets
      - --verbosity=info
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

Lo que cambié aquí y por qué:

- **Rutas relativas en `volumes`** (`./execution`, `./jwt.hex`) en vez de `/home/carlos/...` absolutas. Como ya hiciste `cd ~/ethereum-mainnet` antes de `docker compose up`, las rutas relativas funcionan igual y el archivo es portable entre cualquier usuario o máquina.
- **Quité el bloque `ports:`** de ambos servicios: con `network_mode: host`, Docker ignora por completo cualquier mapeo de `ports:` — el contenedor ya comparte la pila de red del host directamente. Dejarlo ahí no rompe nada, pero engaña al lector haciéndole pensar que ahí se controla qué está expuesto; lo que realmente controla la exposición es UFW (Step 7).
- **Imagen de Prysm actualizada** a `gcr.io/offchainlabs/prysm/beacon-chain`. Prysm se fusionó con Offchain Labs y están migrando todos los repos activos fuera de `prysmaticlabs`; las URLs antiguas (`gcr.io/prysmaticlabs/...`) siguen funcionando hoy pero quedarán deprecadas y dejarán de actualizarse.
- **Quité `--supernode`**: convierte tu nodo en un "super node" que custodia el 100% de las data columns post-Fusaka — aumenta significativamente los requisitos de RAM y ancho de banda, muy por encima de los 16GB/600Mbps de la tabla de hardware de arriba. No lo necesitas para staking normal en solitario; solo tiene sentido si vas a servir blobs históricos a terceros.

## Step 6. ▶️ Run Geth & Prysm Nodes

```bash
docker compose up -d        # start both containers, detached
docker compose logs -f      # tail logs from both containers
docker compose down         # stop and remove containers (before updating)
```

## Step 7. 🔥 UFW

```bash
sudo ufw allow OpenSSH

# Geth P2P
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp

# Prysm P2P (default ports, separate from the RPC/gateway ports 4000/3500)
sudo ufw allow 13000/tcp
sudo ufw allow 12000/udp

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

```bash
sudo ss -tulnp          # ports currently in use
sudo ufw status verbose # active UFW rules
```

## Step 8. 🔄 Checking If Nodes are Synced

**Execution Node (Geth):**

```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```

✅ Fully synced: `{"jsonrpc":"2.0","id":1,"result":false}`
🚫 Still syncing: returns an object with `currentBlock`, `highestBlock`, `startingBlock`.

**Beacon Node (Prysm):**

```bash
curl http://localhost:3500/eth/v1/node/syncing
```

✅ Fully synced: `is_syncing: false`, `sync_distance: "0"`.
🚫 Still syncing: `is_syncing: true`, with `sync_distance` indicating slots behind.

## Step 9. 🔎 Getting the RPC Endpoints

- **Execution (Geth):** `http://127.0.0.1:8545`
- **Beacon (Prysm):** `http://127.0.0.1:3500`

---

# 🟣 Part 2 — Aztec L2 Node

Pick the path that matches what you want to run: a plain **full node** (sync and query the network, no staking) or a **sequencer/validator** (participate in consensus, requires keys and funded accounts).

## Step 1. ⚙️ Install the Aztec CLI

```bash
bash -i <(curl -s https://install.aztec.network)
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
aztec --version
aztec-up latest
```

> Versión de imagen: la documentación oficial usa `aztecprotocol/aztec:2.1.4` como ejemplo en el momento de escribir esto. Aztec testnet se actualiza con frecuencia — usa `aztec-up latest` para resolver siempre la versión recomendada actual en vez de fiarte de un tag fijo copiado de una guía (incluida esta).

## 🟢 Step 2A. Full Node (no staking)

Just syncing and querying the network — no validator keys needed.

```bash
mkdir -p ~/aztec-fullnode/data
cd ~/aztec-fullnode
nano .env
```

```bash
DATA_DIRECTORY=./data
LOG_LEVEL=info
ETHEREUM_HOSTS=http://127.0.0.1:8545
L1_CONSENSUS_HOST_URLS=http://127.0.0.1:3500
P2P_IP=<your external/public IP>
P2P_PORT=40400
AZTEC_PORT=8080
```

```bash
nano docker-compose.yml
```

```yaml
services:
  aztec-full-node:
    image: aztecprotocol/aztec:2.1.4
    container_name: aztec-full-node
    network_mode: host
    restart: unless-stopped
    env_file: .env
    volumes:
      - ./data:/data
    entrypoint: >-
      node
      --no-warnings
      /usr/src/yarn-project/aztec/dest/bin/index.js
      start
      --node
      --archiver
      --network testnet
```

```bash
docker compose up -d
docker compose logs -f
```

Firewall: only `P2P_PORT` (40400 tcp/udp) needs to be open — see [Combined Firewall Reference](#-combined-firewall-reference).

## 🟣 Step 2B. Sequencer / Validator (Production)

This is the path for solo staking. It replaces two inconsistent methods from the original draft — a plaintext private key directly in `docker-compose.yml`, and a `geth account import`-based keystore that silently dropped the BLS key (without it, an attester literally cannot attest — the node would run but never validate). Both are gone; this is the current officially-documented flow.

### Generate your validator keystore

```bash
mkdir -m 700 -p ~/aztec-sequencer/keys ~/aztec-sequencer/data
cd ~/aztec-sequencer

aztec validator-keys new \
  --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000 \
  --data-dir ./keys \
  --file validators.json
```

This single command generates **both** the Ethereum (`eth`) and BLS (`bls`) keys your attester needs — the BLS key cannot be created via `geth account import`, it's an Aztec-specific key type with no Ethereum equivalent. It prints a 12-word mnemonic once: **write it on paper, store it offline.** It's the only way to regenerate these exact keys later.

The resulting `~/aztec-sequencer/keys/validators.json` looks like:

```json
{
  "schemaVersion": 1,
  "validators": [
    {
      "attester": {
        "eth": "0x...",
        "bls": "0x..."
      },
      "feeRecipient": "0x0000000000000000000000000000000000000000000000000000000000000000"
    }
  ]
}
```

> 🔐 **This file contains plaintext private keys.** It is the equivalent of a wallet seed phrase. Lock it down immediately:

```bash
chmod 700 ~/aztec-sequencer/keys
chmod 600 ~/aztec-sequencer/keys/validators.json
```

For production beyond testnet experimentation, Aztec supports remote signers (Web3Signer) for the Ethereum side of the key and JSON V3 encrypted keystores — see [Key storage methods](https://docs.aztec.network/operate/operators/keystore/storage-methods) in the official docs. Note that BLS keys specifically can never be handed off to a remote signer; they always live as a private key on disk, protected by filesystem permissions.

### Fund your attester address

Read the generated address and fund it via the Sepolia faucet:

```bash
jq -r '.validators[0].attester.eth' ~/aztec-sequencer/keys/validators.json
```

Use https://sepolia-faucet.pk910.de/ to send Sepolia ETH to that address — it covers gas for publishing blocks (the original draft's "create a wallet in MetaMask first" step is optional now: the CLI generates a fresh, correctly-formatted key for you, you just need to fund the address it outputs).

### Configure `.env`

```bash
nano .env
```

```bash
DATA_DIRECTORY=./data
KEY_STORE_DIRECTORY=./keys
LOG_LEVEL=info
ETHEREUM_HOSTS=http://127.0.0.1:8545
L1_CONSENSUS_HOST_URLS=http://127.0.0.1:3500
P2P_IP=<your external IP address>
P2P_PORT=40400
AZTEC_PORT=8080
AZTEC_ADMIN_PORT=8880
```

### Configure `docker-compose.yml`

```bash
nano docker-compose.yml
```

```yaml
services:
  aztec-sequencer:
    image: aztecprotocol/aztec:2.1.4
    container_name: aztec-sequencer
    network_mode: host
    restart: unless-stopped
    user: "1000:1000"
    env_file: .env
    volumes:
      - ./data:/var/lib/data
      - ./keys:/var/lib/keystore
    environment:
      KEY_STORE_DIRECTORY: /var/lib/keystore
      DATA_DIRECTORY: /var/lib/data
    entrypoint: >-
      node
      --no-warnings
      /usr/src/yarn-project/aztec/dest/bin/index.js
      start
      --node
      --archiver
      --sequencer
      --network testnet
```

> `user: "1000:1000"` debe coincidir con el UID/GID real del usuario que creó `~/aztec-sequencer/keys` — confírmalo con `id -u` / `id -g` y ajusta si tu primer usuario no es 1000 (poco común, pero pasa en algunas distros o si ya tenías otros usuarios creados). Si el UID no coincide, el contenedor no podrá leer `validators.json` por permisos y fallará al arrancar.

Verify ownership and permissions before starting:

```bash
sudo chown -R 1000:1000 ~/aztec-sequencer/data ~/aztec-sequencer/keys
```

```bash
docker compose up -d
docker compose logs -f
```

Updating the node:

```bash
aztec-up latest
docker compose pull
docker compose up -d
```

To fully reset and resync from scratch:

```bash
docker compose down
rm -rf ~/aztec-sequencer/data/*
docker compose up -d
```

### 🌐 Public IP, Port Forwarding & CG-NAT

Many ISPs use Carrier-Grade NAT (CG-NAT) to conserve IPv4 addresses — under CG-NAT your router doesn't have a true public IP, so incoming connections (including P2P discovery) cannot reach your node no matter how you configure your own firewall. If port forwarding doesn't work after following the steps below, ask your ISP for a dedicated public/static IP.

```bash
curl ipv4.icanhazip.com   # your public IPv4
hostname -I               # your local/internal IPs
```

Router admin panel is usually at `192.168.1.1` or `192.168.0.1` → Port Forwarding / Virtual Server / NAT section. Forward the external `P2P_PORT` (40400) to your machine's internal IP on the same port. **Never forward `AZTEC_ADMIN_PORT` (8880).**

If running multiple nodes on different machines on the same LAN, give each one a static IP or a DHCP reservation (find each machine's MAC via `ip a`) so your forwarding rules don't silently break when DHCP reassigns addresses.

```bash
sudo lsof -i :40400                          # what's listening locally
sudo ss -tulnp | grep 8080                   # confirm AZTEC_PORT is bound
nc -vz localhost 40400                       # local loopback check
nc -vz <your-public-ip> 40400                # external reachability check
nc -vz <peer-internal-ip> 40400              # LAN-level reachability check
```

---

## 🧰 Optional: Developer Tooling

```bash
# Foundry — smart contract development toolkit
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup

# Aztec monorepo (source code, for advanced debugging)
git clone https://github.com/AztecProtocol/aztec-packages.git
cd aztec-packages

# Yarn
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update
sudo apt install -y yarn
yarn -v
```

## 🆔 Verify Node's Peer ID

```bash
docker logs aztec-sequencer 2>&1 | grep -i "peerId" | grep -o '"peerId":"[^"]*"' | cut -d'"' -f4 | head -n 1
```

> Simplificado: el comando original filtraba contenedores por `ancestor=aztecprotocol/aztec:alpha-testnet`, un tag que no coincide con ninguna de las imágenes usadas en esta guía (`2.1.4`). Como el `docker-compose.yml` ya fija `container_name: aztec-sequencer`, referenciarlo directamente es más simple y no se rompe cada vez que cambias de versión.

Search your Peer ID on [Nethermind Explorer](https://aztec.nethermind.io/) — it can take a few hours to appear after your node fully syncs. Replace `aztec-full-node` in the command above if you're running the full-node-only setup instead.

## 🎓 Getting Apprentice Role

Head to the Aztec Discord, `operator | start-here` channel, run `/operator help`.

```bash
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":67}' \
  http://localhost:8080 | jq -r ".result.proven.number"
```

Replace `http://localhost:8080` with your VPS IP:8080 if running remotely. Save the resulting `BLOCK_NUMBER`, then:

```bash
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"node_getArchiveSiblingPath","params":["BLOCK_NUMBER","BLOCK_NUMBER"],"id":67}' \
  http://localhost:8080 | jq -r ".result"
```

Save the returned `PROOF`. Back in `operator | start-here`, run `/operator start`, supplying your sequencer EVM address, the `BLOCK_NUMBER`, and the `PROOF` from above. Once registered, check your status with `/operator my-stats`.

> Note: there's a daily registration quota — if you miss it, try again the next day.

---

## 🔥 Combined Firewall Reference

If you're running both stacks on the same machine, this is the full picture in one place:

| Port | Protocol | Service | Action |
|---|---|---|---|
| 22 | tcp | SSH | `ufw allow OpenSSH` |
| 30303 | tcp/udp | Geth P2P | allow |
| 13000 | tcp | Prysm P2P | allow |
| 12000 | udp | Prysm P2P | allow |
| 40400 | tcp/udp | Aztec P2P | allow |
| 8080 | tcp | Aztec node API | allow only if you need external access; otherwise leave closed |
| **8880** | tcp | **Aztec admin API** | **never allow — no auth on this endpoint** |
| 8545, 8551, 4000, 3500 | tcp | Geth/Prysm local RPC | keep closed externally — these are for `127.0.0.1` / Docker-internal use only |

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

## 🗂️ .gitignore

If you publish your deployment config alongside this guide, add this **before** your first commit:

```
*.env
jwt.hex
password.txt
validators.json
keys/
data/
execution/
consensus/
*.log
```

---

<div align="center">

### 📝 Summary of changes from the original draft

Prysm image migrated to `gcr.io/offchainlabs/prysm/beacon-chain` (org migration) · removed `--supernode` (exceeds the documented hardware budget) · removed non-functional `ports:` blocks under `network_mode: host` · hardcoded `carlos`/`/home/carlos` paths generalized · `apt-get remove` loop fixed to run non-interactively · **critical fix:** the `geth account import`–based attester keystore (missing the required BLS key entirely) replaced with `aztec validator-keys new`, the current officially-documented method · `AZTEC_ADMIN_PORT` (8880) explicitly called out as never-expose across three separate sections · added `.gitignore` and a security checklist, since this guide handles real private keys

</div>
