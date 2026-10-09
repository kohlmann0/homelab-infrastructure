# Homelab ChatGPT Context

**Last updated:** 2026-10-09  
**Purpose:** Authoritative quick-reference for future ChatGPT conversations about this homelab.

> This file is intended to let ChatGPT understand the current homelab without reconstructing the entire conversation history.
>
> **Do not put passwords, API keys, tokens, private certificates, or other secrets in this file.**

---

# 1. High-Level Architecture

The homelab is built around:

- **Ubiquiti UniFi Dream Machine Pro (UDM Pro)**
  - Primary router/network gateway
  - LAN DHCP/DNS
  - Internal DNS records for homelab services
- **Ubuntu Server**
  - Primary Docker host
  - Current LAN IP: `192.168.20.130`
  - Wireless interface: `wlp3s0`
  - Ethernet interface `enp2s0` is currently disconnected
- **Secondary Server: `robotlab.iot`**
  - Same LAN as the main homelab network
  - Current LAN IP: `192.168.20.140`
  - Managed by the Portainer instance running on `homelab.iot` (`192.168.20.130`)
  - Reserved for planned AI workloads and additional automation/edge services
- **Docker**
- **Portainer**
  - Used to deploy/manage Docker Compose stacks
  - Several stacks are deployed from Git repositories
  - A Portainer instance on `homelab.iot` is also controlling the second server at `robotlab.iot`
- **Traefik**
  - Reverse proxy
  - Docker provider
- **Home Assistant**
- **ESPHome**
- **Wyze camera bridge**
  - Used to bring a Wi‑Fi camera stream into Home Assistant via RTSP/bridge tooling
- **LocalLama / LocalLLaMA placeholder**
  - Planned AI container workload for the second server
  - Placeholder has been added, but the container has not been started yet
- Other homelab services may be added later.

Current conceptual layout:

```text
                         Internet
                            |
                         UDM Pro
                            |
                     LAN 192.168.20.0/24
                            |
              +---------+-------------------+
              |                             |
       homelab.iot                robotlab.iot
       192.168.20.130              192.168.20.140
              |                             |
         Portainer (controls both hosts) 
              |                             |
              +---------+-------------------+
                        |
                     Docker
                        |
          +-------------+-------------+
          |             |             |
       Traefik     Home Assistant   ESPHome
       :80/:443        :8123          :6052
          |             |
          +---- Portainer
          +---- Wyze camera bridge
          +---- other services

                         Planned AI workload
                               LocalLama
                               robotlab.iot
```

---

# 2. Network

## LAN

```text
Network: 192.168.20.0/24
Ubuntu Server (homelab.iot): 192.168.20.130
Second server (robotlab.iot): 192.168.20.140
```

Do **not** assume older references to `192.168.20.30` are current. That address was used during earlier troubleshooting but is not the Ubuntu server's current IP.

The second server, `robotlab.iot`, is on the same LAN and is controlled by the Portainer instance running on `homelab.iot` at `192.168.20.130`.

Ubuntu interfaces observed:

```text
wlp3s0
  192.168.20.130/24

enp2s0
  NO-CARRIER / DOWN

docker0
  172.17.0.1/16

Docker bridge networks also exist in:
  172.18.0.0/16
  172.19.0.0/16
```

## Internal DNS

UDM Pro provides internal DNS records pointing to the primary Ubuntu server and the second server:

```text
ha.homelab.iot        -> 192.168.20.130
esphome.homelab.iot   -> 192.168.20.130
portainer.homelab.iot -> 192.168.20.130
traefik.homelab.iot   -> 192.168.20.130
robotlab.iot          -> 192.168.20.140
```

These hostnames are intended for LAN/internal use.

---

# 3. Directory Structure

The homelab services use `/opt/homelab` as the main host directory.

Current important paths:

```text
/opt/homelab/
├── homeassistant/
│   └── config/
├── esphome/
│   └── config/
├── portainer/
│   └── ...
└── ...
```

Traefik configuration is currently under:

```text
/opt/traefik/config
```

Important deployment convention:

**Portainer deploys Git-backed stacks.**

Therefore, Docker Compose files should generally use explicit absolute host paths rather than relying on relative paths such as `./config`.

Example:

```yaml
volumes:
  - /opt/homelab/homeassistant/config:/config
```

---

# 4. Portainer

Portainer is used to manage Docker and deploy stacks.

The main Portainer instance is running on `homelab.iot` (`192.168.20.130`) and is also controlling the second Docker host at `robotlab.iot` (`192.168.20.140`).

Stacks may be deployed from Git repositories.

This means:

- The Git repository contains Compose/infrastructure definitions.
- Runtime application data should generally remain outside the Git repository.
- Avoid committing secrets or runtime databases/state to Git.
- Absolute host paths are preferred for bind mounts because Portainer's Git checkout working directory is not necessarily the host directory where runtime data lives.

Portainer's web interface has previously been exposed on Docker ports including:

```text
8000
9000
9443
```

Portainer intentionally sends a Content Security Policy including:

```text
frame-ancestors 'none'
```

Therefore, embedding the Portainer UI in a Home Assistant iframe/card does not work normally.

Preferred approach: use a normal link/button to open Portainer rather than weakening Portainer's CSP.

---

## Additional host and workload notes

- **Secondary Docker host:** `robotlab.iot`
  - IP: `192.168.20.140`
  - Same LAN/subnet as the primary homelab network
  - Managed by Portainer on `homelab.iot`
- **Planned AI workload:** LocalLama / LocalLLaMA placeholder
  - Intended for the second server
  - Container placeholder is present, but the service has not been started yet
- **Home Assistant camera input:** Wyze camera bridge
  - Added as a Docker container for bringing a Wi‑Fi camera RTSP feed into Home Assistant
  - Purpose is to stream a camera/RTSP source into Home Assistant for automations and viewing

Do not assume the second host is already running the AI workload. The LocalLama placeholder exists, but the service itself is not yet active.

---

# 5. Traefik

Traefik is the chosen reverse proxy.

Current image:

```text
traefik:v3.1
```

Traefik uses the Docker provider.

Current important configuration concepts:

```text
Docker provider enabled
exposedByDefault=false

HTTP entrypoint:
  web -> :80

HTTPS entrypoint:
  websecure -> :443

Temporary dashboard:
  :8080
```

During troubleshooting, the dashboard has been enabled using:

```text
--api.dashboard=true
--api.insecure=true
```

and:

```text
8080:8080
```

**Security note:** the insecure dashboard exposure should be considered temporary. Once the configuration is stable, the dashboard should either be removed/disabled or protected and exposed through an appropriate authenticated HTTPS route.

Traefik mounts:

```text
/var/run/docker.sock:/var/run/docker.sock:ro
/opt/traefik/config:/etc/traefik
```

Traefik uses:

```yaml
security_opt:
  - no-new-privileges:true
```

A Docker `host-gateway` mapping was added to allow the Traefik container to address the Ubuntu host:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

---

# 6. Home Assistant

## Container

Container name:

```text
homeassistant
```

Image:

```text
ghcr.io/home-assistant/home-assistant:stable
```

Network mode:

```text
host
```

Home Assistant therefore listens directly on the Ubuntu host network.

Port:

```text
8123
```

Host configuration:

```text
/opt/homelab/homeassistant/config
```

Container configuration path:

```text
/config
```

Other mounts:

```text
/run/dbus:/run/dbus:ro
/etc/localtime:/etc/localtime:ro
```

The container currently uses:

```yaml
privileged: true
```

## Current working Traefik labels

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.ha.rule=Host(`ha.homelab.iot`)"
  - "traefik.http.routers.ha.entrypoints=web"
  - "traefik.http.routers.ha.service=ha"
  - "traefik.http.services.ha.loadbalancer.server.port=8123"
  - "traefik.http.services.ha.loadbalancer.server.url=http://192.168.20.130:8123"
```

Both `server.port` and explicit `server.url` are currently being retained because this is the known-working configuration.

Do not assume one can be removed without testing.

## Home Assistant Reverse Proxy

Home Assistant initially returned a reverse-proxy/trusted-proxy error even though `configuration.yaml` contained:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.16.0.0/12
    - 192.168.0.0/16
    - 10.0.0.0/8
```

The configuration did not appear to take effect for the actual Traefik proxy.

The decisive fix was manually adding the reverse proxy through the Home Assistant UI:

```text
Settings
  -> Network
     -> Reverse Proxy
```

After adding Traefik there, Home Assistant began working through:

```text
http://ha.homelab.iot
```

This is an important known troubleshooting fact.

---

# 7. ESPHome

## Container

Container name:

```text
esphome
```

Image:

```text
ghcr.io/esphome/esphome:stable
```

Network mode:

```text
host
```

ESPHome dashboard port:

```text
6052
```

Host configuration:

```text
/opt/homelab/esphome/config
```

Container configuration:

```text
/config
```

Device mappings:

```text
/dev/ttyUSB0:/dev/ttyUSB0
/dev/ttyUSB1:/dev/ttyUSB1
```

Uses:

```yaml
privileged: true
```

and:

```text
/etc/localtime:/etc/localtime:ro
```

## Current Traefik labels

```yaml
labels:
  - "traefik.enable=true"

  - "traefik.http.routers.esphome.rule=Host(`esphome.homelab.iot`)"
  - "traefik.http.routers.esphome.entrypoints=web"
  - "traefik.http.routers.esphome.service=esphome"

  - "traefik.http.services.esphome.loadbalancer.server.port=6052"
  - "traefik.http.services.esphome.loadbalancer.server.url=http://192.168.20.130:6052"
```

### IMPORTANT

The ESPHome `server.url` above was:

```text
192.168.20.30
```

This is an old/wrong IP.

The actual Ubuntu server IP is:

```text
192.168.20.130
```

ESPHome appeared to work despite this, which has not yet been fully investigated.

Do not assume the old URL is actually being used by Traefik. It should be tested before making additional changes.

---

# 8. ESPHome Portainer Environment Variables

ESPHome currently uses:

```yaml
environment:
  - USERNAME=${ESPHOME_USERNAME}
  - PASSWORD=${ESPHOME_PASSWORD}
```

The intent is to have Portainer/Compose substitute the values at Stack deployment time.

---

# 9. Current Known-Good HA + ESPHome Compose

The following is the current working reference as provided by the user:

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    volumes:
      - /opt/homelab/homeassistant/config:/config
      - /run/dbus:/run/dbus:ro
      - /etc/localtime:/etc/localtime:ro
    privileged: true
    network_mode: host
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ha.rule=Host(`ha.homelab.iot`)"
      - "traefik.http.routers.ha.entrypoints=web"
      - "traefik.http.routers.ha.service=ha"
      - "traefik.http.services.ha.loadbalancer.server.port=8123"
      - "traefik.http.services.ha.loadbalancer.server.url=http://192.168.20.130:8123"

  esphome:
    container_name: esphome
    image: ghcr.io/esphome/esphome:stable
    restart: always
    privileged: true
    volumes:
      - /opt/homelab/esphome/config:/config
      - /etc/localtime:/etc/localtime:ro
    network_mode: host
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
      - /dev/ttyUSB1:/dev/ttyUSB1
    environment:
      - USERNAME=${ESPHOME_USERNAME}
      - PASSWORD=${ESPHOME_PASSWORD}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.esphome.rule=Host(`esphome.homelab.iot`)"
      - "traefik.http.routers.esphome.entrypoints=web"
      - "traefik.http.routers.esphome.service=esphome"
      - "traefik.http.services.esphome.loadbalancer.server.port=6052"
      - "traefik.http.services.esphome.loadbalancer.server.url=http://192.168.20.130:6052"
```

**Note:** This is the user's current reference configuration, not necessarily the final ideal configuration. In particular, the ESPHome IP needs investigation.

---

# 10. Important Troubleshooting History

## Traefik 404

Initial HA/ESPHome Traefik routing returned 404.

Investigation showed:

- Traefik dashboard was accessible.
- HA labels were initially discovered as `ha@docker`.
- TLS/entrypoint configuration was confusing the initial setup.
- Simplifying to HTTP on the `web` entrypoint was useful during troubleshooting.

Current routing uses:

```text
entrypoint=web
```

and does not currently require TLS for the internal test setup.

## `ha@docker` disappeared

When using:

```yaml
traefik.http.services.ha.loadbalancer.server.url=...
```

without the expected port configuration, Traefik logged:

```text
service "ha" error: port is missing
```

and `ha@docker` disappeared from the dashboard.

Adding:

```yaml
traefik.http.services.ha.loadbalancer.server.port=8123
```

caused the router/service to reappear.

The same principle applies to ESPHome:

```text
server.port=6052
```

## 502 Bad Gateway

Once `ha@docker` appeared again, HA returned:

```text
502 Bad Gateway
```

Testing from inside the Traefik container showed:

```text
wget: can't connect to remote host (192.168.20.30): Host is unreachable
```

This led to discovering that the Ubuntu server was actually:

```text
192.168.20.130
```

not:

```text
192.168.20.30
```

## 400 Bad Request / HA reverse proxy

After correcting the host-gateway configuration, requests reached Home Assistant, but HA initially returned:

```text
400 Bad Request
```

Home Assistant logs showed the request was arriving through an untrusted reverse proxy.

Adding the proxy manually through:

```text
Settings
-> Network
-> Reverse Proxy
```

resolved the issue.

This is currently considered the known-good procedure.

---

# 11. Reverse Proxy Architecture

Current intended architecture:

```text
Browser
   |
   | http://ha.homelab.iot
   v
UDM Pro DNS
   |
   | 192.168.20.130
   v
Traefik
   |
   +---- Home Assistant :8123
   |
   +---- ESPHome :6052
   |
   +---- Portainer
   |
   +---- future services
```

The long-term goal is to use friendly internal hostnames and eventually move toward secure HTTPS.

Do not introduce HTTPS/certificates while basic HTTP routing is being diagnosed.

For internal-only `.iot` hostnames, a public CA such as Let's Encrypt generally cannot simply issue certificates for arbitrary private DNS names. Future HTTPS options may include:

- A real owned domain with split DNS.
- An internal CA.
- Another appropriate internal certificate strategy.

---

# 12. Security Goals

The user wants a secure, professional homelab rather than simply making services work.

Important goals:

- Minimize directly exposed Docker ports.
- Use Traefik as the common reverse proxy.
- Keep management interfaces restricted.
- Avoid exposing Portainer unnecessarily.
- Do not expose Traefik's insecure dashboard permanently.
- Keep Docker socket access carefully controlled.
- Avoid putting secrets in Git.
- Use Git for infrastructure definitions and configuration where appropriate.
- Keep runtime state/databases/secrets separate from source-controlled infrastructure.
- Eventually implement stronger internal HTTPS and authentication.
- Consider defensive monitoring/security tooling for the LAN and Docker host.

---

# 13. Git / Infrastructure Practices

Preferred organization:

```text
homelab/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── NETWORK.md
│   ├── SERVICES.md
│   ├── SECURITY.md
│   ├── OPERATIONS.md
│   └── TROUBLESHOOTING.md
└── stacks/
    ├── traefik/
    ├── homeassistant/
    ├── esphome/
    └── portainer/
```

Infrastructure-as-code should be source controlled.

Avoid committing:

- Passwords
- API keys
- Tokens
- Private keys
- Runtime databases
- Portainer runtime data
- Other secrets

Use Portainer/environment-secret mechanisms or another secure secret-management approach.

---

# 14. Troubleshooting Preferences

The user strongly prefers:

**One troubleshooting step at a time.**

When diagnosing an issue:

1. Give one concrete command/change.
2. Have the user report the result.
3. Use that result to determine the next step.
4. Avoid presenting a long list of simultaneous tests or configuration changes.
5. Keep explanations clear and concise.
6. Do not make multiple unrelated configuration changes at once.

When there is uncertainty, prefer inspecting the actual running Docker/Traefik configuration before changing architecture.

Useful commands already used successfully:

```bash
docker inspect homeassistant --format '{{json .Config.Labels}}'
```

```bash
docker inspect esphome --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -E 'USERNAME|PASSWORD'
```

```bash
docker logs traefik --tail 100
```

```bash
sudo ss -lntp | grep 8123
```

```bash
ip addr
```

```bash
hostname -I
```

---

# 15. Current Open Items

1. **ESPHome Traefik backend URL**
   - Docker Compose IP was `192.168.20.30:6052`.
   - Correct Ubuntu IP is `192.168.20.130`.
   - Determine why ESPHome appears to work despite the old URL before changing it.

2. **robotlab.iot / second server orchestration**
   - `robotlab.iot` is on the same LAN at `192.168.20.140`.
   - It is managed by the Portainer instance on `homelab.iot` (`192.168.20.130`).
   - The LocalLama placeholder has been added, but the container has not been started yet.

3. **Wyze camera bridge integration**
   - A Docker container for Wyze camera bridge has been added for RTSP/streaming integration into Home Assistant.
   - Verify the bridge configuration and camera stream path is stable before expanding automations or exposing routes.

4. **Traefik HTTPS**
   - HTTP routing is the current working baseline.
   - HTTPS/certificate architecture remains to be designed.

5. **Traefik dashboard security**
   - `api.insecure=true` / port 8080 should not remain exposed indefinitely.

6. **Overall homelab defensive security**
   - User is interested in a defensive/white-hat AI security stack around:
     - UDM Pro
     - Ubuntu
     - Docker
     - Home Assistant
     - Other LAN/IoT devices.

---

# 16. Important "Do Not Assume" Items

- Do not assume the Ubuntu server is `192.168.20.30`. Current IP is `192.168.20.130`.
- Do not assume Home Assistant's YAML `trusted_proxies` alone is sufficient; the working configuration required adding the proxy through the HA UI.
- Do not assume Traefik can infer the correct backend for host-networked services without explicit port configuration.
- Do not assume the ESPHome `server.url` is actually being used just because it appears in Compose; verify Traefik's effective configuration if needed.
- Do not make several troubleshooting changes simultaneously.
- Do not introduce HTTPS as part of an HTTP routing diagnosis.
