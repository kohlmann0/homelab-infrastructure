# homelab-infrastructure  
Back up of my home server configurations and docker files  
  
  
Project Structure:  
homelab/  
├── README.md                    # High-level overview  
├── docs/  
│   &emsp;├── ARCHITECTURE.md          # Network + services + dependencies  
│   &emsp;├── NETWORK.md               # IPs, VLANs, DNS, ports  
│   &emsp;├── SERVICES.md              # Docker/Portainer stacks  
│   &emsp;├── SECURITY.md              # Firewall, proxy, trusted proxies  
│   &emsp;├── OPERATIONS.md            # How to deploy/restart/backup  
│   &emsp;└── TROUBLESHOOTING.md       # Known issues and solutions  
└── stacks/  
    &emsp;&emsp;├── traefik/  
    &emsp;&emsp;├── homeassistant/  
    &emsp;&emsp;├── esphome/  
    &emsp;&emsp;└── portainer/  
