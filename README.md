# Azure VPN Gateway Point-to-Site (P2S) Practice Lab using macOS

# 📘 Introduction

Point-to-Site (P2S) VPN allows individual client devices such as laptops or administrator machines to securely connect to Azure Virtual Network over the internet.

This lab demonstrates:

* Azure VPN Gateway
* Point-to-Site VPN
* OpenVPN Configuration
* Certificate Authentication
* macOS VPN Connectivity
* Private VM Access using VPN

using Microsoft Azure.

---

# 🔹 Architecture Diagram

```text id="m7sq41"
                    ┌────────────────────┐
                    │    macOS Client    │
                    │ Tunnelblick/OpenVPN│
                    └─────────┬──────────┘
                              │
                       Point-to-Site VPN
                              │
                    ┌─────────▼──────────┐
                    │ Azure VPN Gateway  │
                    │     RouteBased     │
                    └─────────┬──────────┘
                              │
                 ┌────────────┴────────────┐
                 │     Azure VNet          │
                 │     10.0.0.0/16         │
                 └────────────┬────────────┘
                              │
                   ┌──────────▼──────────┐
                   │      App Subnet     │
                   │     10.0.1.0/24     │
                   └──────────┬──────────┘
                              │
                    ┌─────────▼─────────┐
                    │    Azure Linux VM │
                    │    Private Access │
                    └───────────────────┘
```

---

# 🔹 Prerequisites

## Local Machine

* macOS
* Homebrew
* Azure CLI
* OpenSSL
* Tunnelblick/OpenVPN Client

## Azure Resources

* Azure Subscription
* Contributor Access

---

# 🔹 Theory Points to Remember

## What is Point-to-Site VPN?

Point-to-Site VPN allows:

* Individual laptops
* Remote administrators
* Developers
* Remote users

to connect securely to Azure Virtual Network.

---

## P2S vs S2S VPN

| Feature             | Point-to-Site | Site-to-Site          |
| ------------------- | ------------- | --------------------- |
| Connectivity        | Single Device | Entire Office Network |
| VPN Device Required | No            | Yes                   |
| Best Use Case       | Remote Access | Branch Connectivity   |
| Authentication      | Certificates  | IPSec Shared Key      |

---

## VPN Gateway Types

| Type        | Purpose     |
| ----------- | ----------- |
| RouteBased  | Recommended |
| PolicyBased | Legacy      |

✅ Recommended:

```text id="6g60lc"
RouteBased
```

---

## VPN Protocols

| Protocol | macOS Support |
| -------- | ------------- |
| OpenVPN  | Yes           |
| SSTP     | No            |
| IKEv2    | Partial       |

✅ Recommended:

```text id="c1zhfw"
OpenVPN
```

---

## GatewaySubnet Important Point

Azure deploys VPN Gateway internally inside:

```text id="nsj6yw"
GatewaySubnet
```

Rules:

* Name must be exact
* Dedicated subnet only
* Cannot rename later

---

## VPN Client Address Pool

Example:

```text id="7r4v4k"
172.16.0.0/24
```

Purpose:

* Assigns IPs to VPN users
* Must not overlap with:

  * Local Network
  * Azure VNet

---

## Certificate Authentication Flow

```text id="slttmu"
Root Certificate
        ↓
Signs Client Certificate
        ↓
Client Certificate Installed on Mac
        ↓
Azure Validates Trust
        ↓
VPN Connection Established
```

---

# 🔹 Step 1 — Install Azure CLI

```bash id="6crh6o"
brew update
brew install azure-cli
```

Validate:

```bash id="q1u8m4"
az version
```

---

# 🔹 Step 2 — Install OpenSSL

```bash id="8cjlwm"
brew install openssl
```

Validate:

```bash id="lk6n25"
openssl version
```

---

# 🔹 Step 3 — Login to Azure

```bash id="r7mlpw"
az login
```

---

# 🔹 Step 4 — Set Subscription

```bash id="q27gsl"
az account list --output table
```

```bash id="72kyd4"
az account set \
  --subscription "SUBSCRIPTION_NAME"
```

---

# 🔹 Step 5 — Create Resource Group

```bash id="jlwmxz"
az group create \
  --name rg-vpn-demo \
  --location centralindia
```

---

# 🔹 Step 6 — Create Virtual Network

```bash id="ljwm4k"
az network vnet create \
  --resource-group rg-vpn-demo \
  --name vnet-demo \
  --address-prefix 10.0.0.0/16 \
  --subnet-name appsubnet \
  --subnet-prefix 10.0.1.0/24
```

---

# 🔹 Step 7 — Create GatewaySubnet

```bash id="r34n4h"
az network vnet subnet create \
  --resource-group rg-vpn-demo \
  --vnet-name vnet-demo \
  --name GatewaySubnet \
  --address-prefixes 10.0.255.0/27
```

---

# 🔹 Step 8 — Create Public IP

```bash id="jlwmj7"
az network public-ip create \
  --resource-group rg-vpn-demo \
  --name vpn-pip \
  --sku Standard
```

---

# 🔹 Step 9 — Create VPN Gateway

> Deployment takes around 30–45 minutes.

```bash id="4pdqrf"
az network vnet-gateway create \
  --resource-group rg-vpn-demo \
  --name MyVPN \
  --public-ip-address vpn-pip \
  --vnet vnet-demo \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1
```

---

# 🔹 Step 10 — Verify VPN Gateway

```bash id="jlwm5m"
az network vnet-gateway show \
  --resource-group rg-vpn-demo \
  --name MyVPN \
  --query provisioningState
```

Expected Output:

```text id="jlwm7m"
Succeeded
```

---

# 🔹 Step 11 — Generate Root Certificate

## Create Root Key

```bash id="jlwm3o"
openssl genrsa -out rootCA.key 2048
```

---

## Create Root Certificate

```bash id="jlwmz9"
openssl req -x509 -new -nodes \
  -key rootCA.key \
  -sha256 \
  -days 1024 \
  -out rootCA.pem
```

Example Values:

```text id="gydm60"
Country Name: IN
State: Maharashtra
City: Pune
Organization: CloudLab
Common Name: AzureP2SRoot
```

---

# 🔹 Step 12 — Upload Root Certificate

## Convert Certificate to Single Line

```bash id="jlwm1m"
CERT=$(awk 'NF {sub(/\r/, ""); printf "%s",$0;}' rootCA.pem)
```

---

## Upload Certificate

```bash id="8dby4t"
az network vnet-gateway root-cert create \
  --gateway-name MyVPN \
  --resource-group rg-vpn-demo \
  --name RootCert \
  --public-cert-data "$CERT"
```

---

# 🔹 Step 13 — Configure Point-to-Site VPN

```bash id="w00nnx"
az network vnet-gateway update \
  --resource-group rg-vpn-demo \
  --name MyVPN \
  --address-prefixes 172.16.0.0/24 \
  --vpn-client-protocols OpenVPN
```

---

# 🔹 Step 14 — Generate Client Certificate

## Create Client Key

```bash id="4kxnvu"
openssl genrsa -out client.key 2048
```

---

## Create CSR

```bash id="jjlwm1"
openssl req -new \
  -key client.key \
  -out client.csr
```

---

## Generate Client Certificate

```bash id="jlwm0r"
openssl x509 -req \
  -in client.csr \
  -CA rootCA.pem \
  -CAkey rootCA.key \
  -CAcreateserial \
  -out client.crt \
  -days 500 \
  -sha256
```

---

# 🔹 Step 15 — Export PFX Certificate

```bash id="ljwm4v"
openssl pkcs12 -export \
  -out client.pfx \
  -inkey client.key \
  -in client.crt
```

---

# 🔹 Step 16 — Generate VPN Client Package

```bash id="7l6j2u"
az network vnet-gateway vpn-client generate \
  --resource-group rg-vpn-demo \
  --name MyVPN
```

---

# 🔹 Step 17 — Download VPN Client Package

Go to:

```text id="jlwm44"
Azure Portal
→ Virtual Network Gateway
→ Point-to-site configuration
→ Download VPN Client
```

Download:

```text id="jlwm9x"
OpenVPN Configuration Package
```

---

# 🔹 Step 18 — Install VPN Client on macOS

## Install Tunnelblick

[Tunnelblick](https://tunnelblick.net?utm_source=chatgpt.com)

OR

## Install OpenVPN Connect

[OpenVPN Connect](https://openvpn.net/client-connect-vpn-for-mac-os/?utm_source=chatgpt.com)

---

# 🔹 Step 19 — Import VPN Configuration

1. Extract ZIP Package
2. Locate `.ovpn` file
3. Open with Tunnelblick/OpenVPN
4. Import VPN Profile

---

# 🔹 Step 20 — Install Client Certificate

## Open Keychain Access

```text id="jlwm6z"
Applications
→ Utilities
→ Keychain Access
```

---

## Import Certificate

* Double-click `client.pfx`
* Enter Password
* Import into:

```text id="jlwm5c"
login keychain
```

---

# 🔹 Step 21 — Connect VPN

1. Open Tunnelblick/OpenVPN
2. Select VPN Profile
3. Click Connect

---

# 🔹 Step 22 — Validate VPN Connection

## Check VPN Interface

```bash id="jlwm8q"
ifconfig
```

Look for:

```text id="jlwm0s"
utun interface
```

---

## Check Routes

```bash id="ljwm0k"
netstat -rn
```

---

# 🔹 Step 23 — Create Test Linux VM

```bash id="jlwm6r"
az vm create \
  --resource-group rg-vpn-demo \
  --name vm-demo \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name vnet-demo \
  --subnet appsubnet
```

---

# 🔹 Step 24 — SSH using Private IP

```bash id="jlwmx0"
ssh azureuser@10.0.1.4
```

---

# 🔹 Useful Commands

## Check VPN Gateway

```bash id="jlwm3x"
az network vnet-gateway list --output table
```

---

## List Root Certificates

```bash id="jlwm2a"
az network vnet-gateway root-cert list \
  --resource-group rg-vpn-demo \
  --gateway-name MyVPN
```

---

## Check Public IP

```bash id="jjlwm3"
az network public-ip list --output table
```

---

## Check VNet

```bash id="jlwm4f"
az network vnet list --output table
```

---

## Delete Lab

```bash id="jlwm1g"
az group delete \
  --name rg-vpn-demo \
  --yes
```

---

# 🔹 Ports Used

| Port | Purpose |
| ---- | ------- |
| 443  | OpenVPN |
| 22   | SSH     |
| 3389 | RDP     |

---

# 🔹 Security Best Practices

* Use Certificate Authentication
* Avoid Public IP on VMs
* Use NSG Restrictions
* Use SSH Keys
* Rotate Certificates Regularly
* Use Least Privilege Access

---

# 🔹 Common Errors & Fixes

## InvalidArgumentValue

Cause:

```text id="jlwm6u"
Certificate uploaded in multi-line format
```

Fix:

```bash id="jlwm8j"
CERT=$(awk 'NF {sub(/\r/, ""); printf "%s",$0;}' rootCA.pem)
```

---

## VPN Gateway Creation Failed

Check:

* GatewaySubnet exists
* Standard Public IP used
* Subscription quota available

---

## VPN Client Not Connecting

Check:

* Client certificate imported
* VPN Gateway provisioned
* Correct OpenVPN package used

---

## Permission Denied

```bash id="jlwm7y"
chmod 600 client.key
```

---

# 🔹 Important Points to Remember

* VPN Gateway deployment takes 30–45 mins
* Use Standard Public IP only
* Use RouteBased VPN only
* OpenVPN recommended for macOS
* GatewaySubnet name is mandatory
* Client certificate must be signed using uploaded root certificate
* Re-download VPN package after VPN configuration changes
* Avoid overlapping IP ranges
* VPN Gateway incurs Azure cost even when idle

---

# 🔹 Interview Questions

## What is Point-to-Site VPN?

Secure VPN connection from individual device to Azure Virtual Network.

---

## Difference between P2S and S2S?

P2S connects single client machine.
S2S connects entire office network.

---

## Why RouteBased VPN?

Supports modern VPN protocols and dynamic routing.

---

## Which VPN protocol recommended for macOS?

```text id="jlwm9p"
OpenVPN
```

---
