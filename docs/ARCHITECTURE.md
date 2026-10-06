# Homelab Architecture

## Network

| Component | Address | Purpose |
|---|---|---|
| UDM Pro | ... | Router/DHCP/DNS |
| Ubuntu Server | 192.168.20.130 | Docker host |
| LAN | 192.168.20.0/24 | Main LAN |

## Internal DNS

| Hostname | Address |
|---|---|
| ha.homelab.iot | 192.168.20.130 |
| esphome.homelab.iot | 192.168.20.130 |
| traefik.homelab.iot | 192.168.20.130 |
| portainer.homelab.iot | 192.168.20.130 |

## Docker

Docker host:
192.168.20.130

### Traefik

Ports:
- 80 → HTTP
- 443 → HTTPS
- 8080 → Dashboard (temporary/insecure)

Network:
- Docker bridge

### Home Assistant

Container:
homeassistant

Network:
host

Port:
8123

Host configuration:
`/opt/homelab/homeassistant/config`

Reverse proxy:
Traefik

Trusted proxy:
Configured through Home Assistant UI

### ESPHome

Container:
esphome

Network:
host

Port:
6052

Host configuration:
`/opt/homelab/esphome/config`

Reverse proxy:
Traefik



## Known Important Details

- HA and ESPHome use Docker host networking.
- Traefik uses the Docker provider.
- Host-networked services require explicit Traefik `server.port`.
- HA required adding the Traefik proxy through:
  Settings → Network → Reverse Proxy.
- Ubuntu server LAN IP is 192.168.20.130.
- Do not assume older references to 192.168.20.30 are still valid.
- Portainer deploys stacks from Git.
- Compose files should use absolute host paths under /opt/homelab.