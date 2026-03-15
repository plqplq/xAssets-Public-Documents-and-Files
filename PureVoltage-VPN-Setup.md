# PureVoltage VPN Setup Guide (IKEv2 - New York)

## Prerequisites

### Disable IP Fragment Blocking

Before starting, check your ISP router/hub/firewall settings and ensure "Block fragmented IP packets" is **disabled**. PureVoltage's IKE_AUTH response is fragmented into multiple UDP packets. If your router drops fragments, the VPN handshake will silently time out on all platforms.

- **Virgin Media Super Hub:** Advanced Settings > Firewall > Block fragmented IP packets > OFF
- **Other routers:** Look in firewall/security settings for fragment blocking or similar

### Credentials

You need your VPN username and password from the PureVoltage control panel.

### IPMI Address

Your server's IPMI/BMC address is on the PureVoltage VPN subnet (10.7.8.0/21). Check your welcome email or control panel for the specific IP (e.g. 10.7.10.107).

---

## Part A: Setting Up the VPN on Ubuntu

This is the only reliable method. Windows cannot connect (see Part C for explanation).

### 1. Install strongSwan and plugins

```bash
sudo apt update
sudo apt install -y charon-systemd libstrongswan-extra-plugins libcharon-extra-plugins
```

### 2. Download Let's Encrypt CA certificates

PureVoltage uses a Let's Encrypt certificate. strongSwan needs the root and intermediate CA certificates to verify it.

```bash
sudo wget -O /etc/swanctl/x509ca/isrg-root-x1.pem https://letsencrypt.org/certs/isrgrootx1.pem
sudo wget -O /etc/swanctl/x509ca/lets-encrypt-r12.pem https://letsencrypt.org/certs/2024/r12.pem
```

> **Note:** The intermediate certificate (R12) may change over time as Let's Encrypt rotates intermediates. If certificate validation fails in future, check https://letsencrypt.org/certificates/ for the current intermediate.

### 3. Create the VPN configuration

```bash
sudo nano /etc/swanctl/conf.d/purevoltage.conf
```

Paste the following, replacing `YOUR-VPN-USERNAME` and `YOUR-VPN-PASSWORD` with your actual credentials:

```
connections {
    purevoltage {
        fragmentation = yes
        remote_addrs = vpn.nyc.purevoltage.com
        vips = 0.0.0.0
        remote {
            auth = pubkey
            id = vpn.nyc.purevoltage.com
        }
        local {
            auth = eap-mschapv2
            id = YOUR-VPN-USERNAME
            eap_id = YOUR-VPN-USERNAME
        }
        children {
            purevoltage {
                remote_ts = 0.0.0.0/0
                start_action = none
            }
        }
    }
}

secrets {
    eap-pv {
        id = "YOUR-VPN-USERNAME"
        secret = "YOUR-VPN-PASSWORD"
    }
}
```

Save with Ctrl+O, Enter, Ctrl+X.

### 4. Connect

```bash
sudo swanctl --load-all
sudo swanctl --initiate --child purevoltage
```

You should see output ending with:

```
IKE_SA purevoltage[x] established between ...
CHILD_SA purevoltage{x} established with SPIs ... and TS 10.7.x.x/32 === 10.7.8.0/21
initiate completed successfully
```

### 5. Add the route to the IPMI subnet

strongSwan assigns a virtual IP but doesn't automatically add a route. Find your assigned VPN IP and network interface:

```bash
ip addr show | grep 10.7
```

Then add the route (replace `br0` with your actual network interface name, e.g. `eth0`, `ens18`, etc.):

```bash
sudo ip route add 10.7.8.0/21 dev br0 src <YOUR-ASSIGNED-VPN-IP>
```

### 6. Verify connectivity

```bash
ping <YOUR-IPMI-IP>
```

### 7. Disconnect

```bash
sudo swanctl --terminate --ike purevoltage
```

### Quick connect script

Save this as `~/vpn-connect.sh` for fast access during outages:

```bash
#!/bin/bash
sudo systemctl restart strongswan
sleep 2
sudo swanctl --load-all
sudo swanctl --initiate --child purevoltage
sleep 2

# Get the assigned VPN IP
VPN_IP=$(ip addr show dev br0 | grep "10\.7\." | awk '{print $2}' | cut -d/ -f1)
if [ -n "$VPN_IP" ]; then
    sudo ip route add 10.7.8.0/21 dev br0 src $VPN_IP 2>/dev/null
    echo "VPN connected. VPN IP: $VPN_IP"
    echo "Route to 10.7.8.0/21 added."
else
    echo "VPN may not have connected. Check output above."
fi
```

```bash
chmod +x ~/vpn-connect.sh
```

---

## Part B: Accessing IPMI from Windows via SSH Tunnel

Since the VPN only works on Ubuntu, Windows users can tunnel through a connected Ubuntu machine using SSH.

### Prerequisites

- An Ubuntu machine with the VPN connected (Part A)
- SSH access to that Ubuntu machine from your Windows PC
- The IPMI IP address (e.g. 10.7.10.107)

### 1. Create the SSH tunnel

Open PowerShell and run:

```powershell
ssh -L 8443:10.7.10.107:443 -L 8080:10.7.10.107:80 user@ubuntu-hostname-or-ip
```

Replace:
- `10.7.10.107` with your IPMI IP
- `user@ubuntu-hostname-or-ip` with your Ubuntu login (e.g. `paul@ub1` or `paul@10.5.0.34`)

This maps:
- `https://localhost:8443` to the IPMI web console (HTTPS)
- `http://localhost:8080` to the IPMI web console (HTTP)

### 2. Access IPMI

Open a browser and navigate to:

```
https://localhost:8443
```

You may need to accept a self-signed certificate warning.

### 3. Quick access script

Save this as `Connect-IPMI.ps1` on the Z: drive for the team:

```powershell
param(
    [string]$UbuntuHost = "paul@10.5.0.34",
    [string]$IPMIIP = "10.7.10.107"
)

Write-Host "Connecting to IPMI ($IPMIIP) via Ubuntu VPN gateway ($UbuntuHost)..." -ForegroundColor Cyan
Write-Host "Once connected, open https://localhost:8443 in your browser" -ForegroundColor Yellow
Write-Host "Press Ctrl+C to disconnect" -ForegroundColor Yellow
Write-Host ""

ssh -L 8443:${IPMIIP}:443 -L 8080:${IPMIIP}:80 $UbuntuHost
```

Usage:

```powershell
# With defaults
.\Connect-IPMI.ps1

# With custom parameters
.\Connect-IPMI.ps1 -UbuntuHost "paul@10.5.0.34" -IPMIIP "10.7.10.107"
```

---

## Part C: Why Windows IKEv2 Cannot Connect

This section documents why the built-in Windows VPN client cannot connect to PureVoltage's IKEv2 VPN, despite their published Windows instructions.

### The Problem

PureVoltage's VPN server requires **DH Group ECP_521** (NIST P-521 elliptic curve) for the IKE_SA_INIT phase (Phase 1) of the IKEv2 handshake.

Windows' built-in IKEv2 client only supports the following DH groups:

| Windows Value | DH Group |
|---------------|----------|
| None | None |
| Group1 | 768-bit MODP |
| Group2 | 1024-bit MODP |
| Group14 | 2048-bit MODP |
| ECP256 | NIST P-256 |
| ECP384 | NIST P-384 |
| Group24 | 2048-bit MODP with 256-bit prime |

**ECP_521 is not in this list.** There is no registry hack, update, or configuration that can add it.

### What Happens

1. Windows offers its supported DH groups during IKE_SA_INIT
2. The server rejects all of them and requests ECP_521
3. Windows cannot comply, so the handshake times out
4. Windows reports: "The network connection between your computer and the VPN server could not be established because the remote server is not responding"

### Evidence

This was confirmed using strongSwan on Ubuntu, which showed the server explicitly requesting ECP_521:

```
[IKE] peer didn't accept DH group ECP_256, it requested ECP_521
```

The final negotiated proposal was:

```
IKE:AES_GCM_16_256/PRF_HMAC_SHA2_512/ECP_521
```

### Additional Issue: PureVoltage's Windows Instructions Are Wrong

Their published IPsec configuration specifies:
- DHGroup: Group14
- Encryption: AES256
- Authentication: SHA256128

But their server actually negotiates:
- DHGroup: ECP_521
- Encryption: AES_GCM_16_256
- PRF: HMAC_SHA2_512

These instructions have never worked and cannot work on any Windows version.

### What PureVoltage Could Fix

If PureVoltage configured their VPN server to also accept **ECP384** or **Group14** for the IKE_SA phase, Windows clients would be able to connect. This is a server-side configuration change.

### Tested Platforms (all failed on Windows)

- Windows Server 2022
- Windows 10
- Tested with and without custom IPsec configuration
- Tested with Let's Encrypt CA certificates manually installed
- Tested with NegotiateDH2048_AES256 registry key enabled

### Platforms Where VPN Works

- Ubuntu 22.04+ with strongSwan (using ECP_521)
- iPhone on mobile data (not on Virgin Media broadband)
- PureVoltage staff confirmed connection from iPhone
