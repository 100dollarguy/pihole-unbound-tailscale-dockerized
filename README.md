## Private DNS Ad-Blocking and Remote Access with Pi-hole + Unbound + Tailscale

# 👋 Why I Built This

I care about online privacy and wanted to block ads and trackers across all my devices, not just in browsers. Public DNS providers like Google or Cloudflare are fast, but often log user data. I wanted full control over DNS resolution and to keep it private.

My internet provider (Jio Fiber) uses CGNAT, so I don’t get a public IP address. That makes remote access tricky. This project uses Tailscale to solve that problem while offering secure DNS resolution and ad-blocking.

# 🔍 Problems Solved

- Block ads and trackers across all devices
- Encrypt and control DNS traffic (no third-party logging)
- Secure, recursive DNS resolution via Unbound
- Enable remote DNS and Pi-hole admin access even behind CGNAT
- Learn real-world Linux, Docker, VPN, DNS, and firewall management

![Architecture](/screenshots/Architecure-Pihole+Docker.png)

# 🧰 Tools Used

- macOS (host)
- Colima to run Docker with Lima backend on macOS
- Docker for containerization
- Pi-hole (ad-blocking DNS)
- Unbound (recursive DNS resolver)
- Tailscale (zero-config WireGuard-based VPN)

# 🛠️ How I Built It

If you are not using a Mac skip to [Section 2](#2-create-docker-compose-setup).

## 1. Setup Docker + Colima

Install Colima and start it:

```shell
brew install colima

colima start --memory 2 --cpu 2 --disk 15
```

Ensure Docker is pointed to Colima:

```shell
export DOCKER_HOST=unix://$HOME/.colima/default/docker.sock
```

## 2. Generate Tailscale Auth Key

Went to Tailscale admin panel, go to "[Access Controls](https://login.tailscale.com/admin/acls/visual/general-access-rules)" and open the Tab "[Tags](https://login.tailscale.com/admin/acls/visual/tags)" 
Created an ACL tag (e.g. tag:pihole).\
⚠️ If yoo use a different Tag, be sure to change it in the [The docker-compose.yml File](docker-compose.yml).

Go to the "[Settings](https://login.tailscale.com/admin/settings/general)" and under "Personal Setting" open "[Keys](https://login.tailscale.com/admin/settings/keys)"
Generated an auth key tagged with tag:pihole
Used that auth key in Docker Compose for Tailscale container.

⚠️ Should your docker compose have an error and your tailscale docker container crash, you might have to create a new Auth Key. You can circumvent this by creating a key that is reusable. Make sure to set the "Expiration" to 1 day. This is less secure as someone getting a hold of your key could add Machines to your Account.

## 3. Create Docker Compose Setup

Create the docker-compose.yml with services based on [The docker-compose.yml File](docker-compose.yml) or download the latest version:
```shell
curl -o https://raw.githubusercontent.com/100dollarguy/pihole-unbound-tailscale-dockerized/refs/heads/master/docker-compose.yml
```

⚠️ Make sure to select the correct platform for each container:
- Mac: linux/arm64
- 64bit OS: linux/amd64
- 32bit OS: linux/386 \(Only [pihole/pihole](https://hub.docker.com/r/pihole/pihole/tags) and [tailscale/tailscale](https://hub.docker.com/r/tailscale/tailscale/tags) offer this Platform. You could go with [madnuttah/unbound](https://hub.docker.com/r/madnuttah/unbound/tags) but will need to adapt the [The docker-compose.yml File](docker-compose.yml)\)

#### Unbound
Unbound is listening on port 5335 by default, the compose file maps its to Port 53.\
Also the config is mounted to a file on the host machine \("unbound-conf/unbound.conf"\)

#### Tailscale
You will need to replace "${AUTHKEY}" with the Key generated in [Section 3](#3-create-docker-compose-setup).\
The data files from tailscale are mounted to the folder "tailscale-data" on the host machine.

#### Pi-hole
Pi-hole depends on the other two containers and is using Unbound as its upstream DNS.
Set your timezone under the Environment to show the correc date and time in pihole.

Create the necessary folders:

```shell
mkdir -p unbound-conf pihole/etc-pihole pihole/etc-dnsmasq.d
```

### 3.1 Prepare Unbound Files

Unbound requires a root server list and DNSSEC trust anchor.

Download them:

#### Root DNS server list

```shell
curl -o ./unbound-conf/root.hints https://www.internic.net/domain/named.cache
```

#### Root DNSSEC trust key

```shell
curl -o ./unbound-conf/root.key https://data.iana.org/root-anchors/root.key
```

#### Unbound config

Create your Unbound config in the "unbound-config" Folder:
```shell
touch ./unbound-config/unbound.conf
```
Enter the configuration options you want to apply. If you are unsure, go with (the official guide)[https://docs.pi-hole.net/guides/dns/unbound/].


## 4. Set ACL Tag Permissions
Go Back to the "[Access Controls](https://login.tailscale.com/admin/acls/visual/general-access-rules)" and click on "[JSON Editor](https://login.tailscale.com/admin/acls/file)". Search for "tagOwners" and add the "acls" sectiona s shown below:

```json
"tagOwners": {
  "tag:pihole": ["user:your_email@example.com"]
},
"acls": [
  {
    "action": "accept",
    "src": ["tag:pihole"],
    "dst": ["*:*"]
  }
]
```

## 5. Start Services

```shell
docker compose up -d
```
The containers should all startup, pihole might take roughly 40s to show up as started and healthy.

The tailscale container auto-joins the network, and Pi-hole is accessible locally.

![Pihole](/screenshots/Pihole-admin-dashboard.png)

### 5.1 (Optional) Test Unbound Resolution

Ensure Unbound is resolving domains before Pi-hole starts using it:

```shell
dig @127.0.0.1 -p 5335 google.com
```

## 6. Add Tailscale Global DNS

Open your Tailscale [DNS settings](https://login.tailscale.com/admin/dns).
Under "Nameservers" and then "Global Domains" add the Pi-hole container’s Tailscale IP as a global nameserver.
Click on the switch named "Override DNS Servers".

![Tailscale](/screenshots/Tailscale-dns.png)

## 7. Remote Access

Tested DNS queries from remote Tailscale-connected devices:

```shell
nslookup google.com 100.x.x.x
```

Expected output:

Name:    google.com  
Address: 142.251.221.110

![nslookup](/screenshots/nslookup.png)


🤯 What I Learned
- Difference between recursive and forwarded DNS
- Tailscale ACLs and DNS overrides
- Docker networking and port mappings
- Unbound configuration for Pi-hole
- Colima/Lima integration for macOS-based Docker usage

![unbound](/screenshots/unbound-logs.png)


🚀 Future Plans

- Add DNS-over-HTTPS fallback
- Setup Headscale server 
- Use Raspberry Pi or always-on NUC
- Integrate 2FA for Pi-hole admin
- Add logging and monitoring for DNS performance


## ✨ License

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

This project is licensed under the [MIT License](LICENSE).
You're free to use, modify, and share it — personally or commercially.

Feel free to fork it, improve it for your own setup, or share with others!
