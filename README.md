# homelab-infrastructure  
Back up of my home server configurations and docker files  
  
  
Project Structure:  
homelab/  
├── README.md                    # High-level overview  
├── docs/  
│   ├── ARCHITECTURE.md          # Network + services + dependencies  
│   ├── NETWORK.md               # IPs, VLANs, DNS, ports  
│   ├── SERVICES.md              # Docker/Portainer stacks  
│   ├── SECURITY.md              # Firewall, proxy, trusted proxies  
│   ├── OPERATIONS.md            # How to deploy/restart/backup  
│   └── TROUBLESHOOTING.md       # Known issues and solutions  
└── stacks/  
    ├── traefik/  
    ├── homeassistant/  
    ├── esphome/  
    └── portainer/  
