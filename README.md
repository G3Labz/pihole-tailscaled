# pihole-tailscaled

A Docker Compose setup that combines Pi-hole with Unbound and Tailscale for secure, network-wide ad blocking with recursive DNS resolution, accessible from anywhere through your Tailscale network.

## Features

- 🛡️ Network-wide ad blocking with Pi-hole
- 🔄 Recursive DNS resolution with Unbound (no third-party DNS dependency)
- 🌐 Secure access via Tailscale network
- 🔒 No need to expose DNS ports to the public internet
- 🔄 Persistent Tailscale state and Pi-hole configuration
- 📊 Web-based admin interface for Pi-hole management
- 🚀 Easy deployment with Docker Compose

## Prerequisites

- Docker and Docker Compose installed
- A Tailscale account and auth key ([Get one here](https://login.tailscale.com/admin/settings/keys))
- Ports 53, 80, and 443 available on the host (or modify the compose file)

## Quick Start

1. **Clone/navigate to the repository:**

   ```bash
   cd pihole-tailscaled
   ```

2. **Copy the example configuration:**

   ```bash
   cp docker-compose.example.yml docker-compose.yml
   ```

3. **Edit `docker-compose.yml` with your settings:**

   ```bash
   nano docker-compose.yml
   ```

   Update the following values:
   - `TS_AUTHKEY`: Your Tailscale auth key (get one from [Tailscale admin](https://login.tailscale.com/admin/settings/keys))
   - `hostname`: Optional - customize the Tailscale machine name
   - `TZ`: Set your timezone
   - `FTLCONF_webserver_api_password`: Set a secure password for the Pi-hole admin interface

4. **Start the services:**

   ```bash
   docker compose up -d
   ```

5. **Check logs:**

   ```bash
   docker compose logs -f
   ```

6. **Access Pi-hole:**
   - Find your Tailscale IP: `tailscale ip` or check the [Tailscale admin console](https://login.tailscale.com/admin/machines)
   - Open `http://<tailscale-ip>/admin` or `https://<tailscale-ip>/admin`

## Configuration

### Environment Variables (Tailscale)

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `TS_AUTHKEY` | Yes | Tailscale authentication key | `tskey-auth-...` |
| `TS_STATE_DIR` | Yes | Directory for Tailscale state | `/var/lib/tailscale` |
| `TS_USERSPACE` | No | Run in userspace mode (recommended for containers) | `true` |
| `TS_EXTRA_ARGS` | No | Additional Tailscale arguments | `--advertise-tags=tag:container` |

### Environment Variables (Pi-hole)

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `TZ` | Recommended | Your timezone | `America/Sao_Paulo` |
| `FTLCONF_webserver_api_password` | Recommended | Admin password for web interface | `your-secure-password` |
| `FTLCONF_dns_listeningMode` | Recommended | DNS listening mode | `ALL` |
| `FTLCONF_dns_upstreams` | No | Upstream DNS servers (semicolon separated) | `1.1.1.1;1.0.0.1` |

### Ports

| Port | Protocol | Service | Description |
|------|----------|---------|-------------|
| `5353` | TCP/UDP | DNS | DNS queries (mapped to container port 53) |
| `8080` | TCP | HTTP | Web interface (HTTP) |
| `8443` | TCP | HTTPS | Web interface (HTTPS) |

> **Note:** Non-privileged ports (5353, 8080, 8443) are used to avoid issues with rootless Docker. Configure your DNS clients to use port 5353.

### Volume Mounts

| Path | Description |
|------|-------------|
| `./tailscale/state` | Persistent Tailscale state |
| `./pihole_data` | Pi-hole configuration and databases |
| `./unbound` | Unbound configuration files |
| `./etc-dnsmasq.d` | Custom dnsmasq configuration (optional) |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Tailscale Network                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  tailscale-pihole container                                  │
│  ├── Tailscale daemon (userspace mode)                       │
│  └── Exposes ports: 53 (DNS), 80, 443 (Web)                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  pihole container (network_mode: service:tailscale-pihole)  │
│  ├── Pi-hole DNS server (ad blocking)                        │
│  └── Pi-hole Web Admin interface                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  unbound container (recursive DNS resolver)                  │
│  ├── Queries root DNS servers directly                       │
│  └── No reliance on third-party DNS (Google, Cloudflare)     │
└─────────────────────────────────────────────────────────────┘
```

## Why Unbound?

Unbound is a recursive DNS resolver that queries the authoritative DNS servers directly, starting from the root servers. Benefits include:

- **Privacy**: No third-party DNS provider sees your queries
- **Independence**: Works even if public DNS providers are down
- **DNSSEC**: Full validation of DNS responses
- **Performance**: Local caching reduces latency for repeated queries

## Using Pi-hole as DNS Server

Once running, you can configure your devices to use Pi-hole as their DNS server:

### Option 1: Per-Device Configuration
Set your device's DNS server to the Tailscale IP of your Pi-hole container.

### Option 2: Tailscale DNS Settings
1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/dns)
2. Add your Pi-hole's Tailscale IP as a Global Nameserver
3. Enable "Override local DNS" if desired

### Option 3: Router Configuration
Configure your router to use Pi-hole as the primary DNS server for your entire network.

## Pi-hole Commands

Execute Pi-hole commands inside the container:

```bash
# Update gravity (block lists)
docker exec pihole pihole updateGravity

# Whitelist a domain
docker exec pihole pihole -w example.com

# Blacklist a domain
docker exec pihole pihole -b ads.example.com

# Check Pi-hole status
docker exec pihole pihole status
```

## Unbound Maintenance

### Update Root Hints

The root hints file should be updated periodically (every 6 months is fine):

```bash
curl -s https://www.internic.net/domain/named.root -o ./unbound/root.hints
docker compose restart unbound
```

### Test Unbound

```bash
# Test Unbound is resolving correctly
docker exec unbound drill @127.0.0.1 -p 5335 cloudflare.com

# Check Unbound logs
docker compose logs unbound
```

## Troubleshooting

### Port 53 Already in Use

If port 53 is already in use (common on systems with systemd-resolved):

```bash
# Check what's using port 53
sudo lsof -i :53

# If systemd-resolved, disable it
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```

Or change the DNS port in the compose file to a different port.

### Cannot Access Admin Interface

1. Verify the container is running: `docker compose ps`
2. Check logs for errors: `docker compose logs pihole`
3. Ensure you're accessing via the Tailscale IP, not localhost

### DNS Not Working

1. Verify Pi-hole is running: `docker exec pihole pihole status`
2. Check if DNS is listening: `docker exec pihole netstat -tuln | grep 53`
3. Test DNS resolution: `dig @<tailscale-ip> google.com`
4. Verify Unbound is healthy: `docker compose ps` (check health status)
5. Test Unbound directly: `docker exec unbound drill @127.0.0.1 -p 5335 google.com`

### Unbound Not Starting

1. Check configuration: `docker exec unbound unbound-checkconf`
2. Review logs: `docker compose logs unbound`
3. Ensure root.hints file exists in `./unbound/`

## Updating

To update Pi-hole to the latest version:

```bash
docker compose pull
docker compose down
docker compose up -d
```

> **Note:** Always read the [Pi-hole release notes](https://github.com/pi-hole/docker-pi-hole/releases) before upgrading, especially for major version changes.

## References

- [Pi-hole Docker Repository](https://github.com/pi-hole/docker-pi-hole)
- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [Tailscale Documentation](https://tailscale.com/kb/)
- [Pi-hole Discourse Forum](https://discourse.pi-hole.net/)

## License

This project is provided as-is for personal use.
