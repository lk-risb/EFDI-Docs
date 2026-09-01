# VPN

Two separate layers are used:

| Layer         | Purpose                                                              |
| ------------- | ------------------------------------------------------------------- |
| **NetBird**   | Assembles the whole network of machines into one overlay.          |
| **WireGuard** | Private link used **only** between the Swarm nodes.         |

Every machine joins NetBird. On Swarm nodes a dedicated WireGuard tunnel is set
up on top for node-to-node swarm traffic.

---

## Part A — NetBird (all machines)

### 1. Install

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sh
```

### 2. Connect

Ask the NetBird admin for both the management URL and a setup key, then join:

```bash
netbird up --management-url <MANAGEMENT-URL> --setup-key <SETUP-KEY>
```

### 3. Verify

```bash
netbird status
```

---

## Part B — WireGuard (Swarm nodes only)

Used exclusively to connect the swarm servers to each other. All other traffic
goes over NetBird.

> [!NOTE]
> Example below is for a two-node swarm using the `10.10.0.0/24` tunnel subnet
> and UDP port `52820`. Add more `[Peer]` blocks for additional nodes.

### 1. Install

```bash
sudo apt install wireguard
```

### 2. Generate keys (on every node)

```bash
wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
sudo chmod 400 /etc/wireguard/privatekey /etc/wireguard/publickey
```

Generate one preshared key that is shared by all nodes:

```bash
wg genpsk
```

Collect from each node: its **public key** and its reachable **endpoint address**
(use the node's NetBird IP) — you need them for the peer config.

### 3. Configure

Create `/etc/wireguard/wg1.conf`.

**Node 1** (`10.10.0.1`):

```ini
[Interface]
Address = 10.10.0.1/24
ListenPort = 52820
PrivateKey = <NODE1_PRIVATE_KEY>

[Peer]
PublicKey = <NODE2_PUBLIC_KEY>
PresharedKey = <COMMON_PRESHARED_KEY>
AllowedIPs = 10.10.0.0/24
Endpoint = <NODE2_ADDRESS>:52820
```

**Node 2** (`10.10.0.2`):

```ini
[Interface]
Address = 10.10.0.2/24
ListenPort = 52820
PrivateKey = <NODE2_PRIVATE_KEY>

[Peer]
PublicKey = <NODE1_PUBLIC_KEY>
PresharedKey = <COMMON_PRESHARED_KEY>
AllowedIPs = 10.10.0.0/24
Endpoint = <NODE1_ADDRESS>:52820
```

### 4. Enable

```bash
sudo systemctl enable --now wg-quick@wg1
```

### 5. Verify

```bash
sudo wg show
ping 10.10.0.1   # from node 2
ping 10.10.0.2   # from node 1
```
