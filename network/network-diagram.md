graph LR
    classDef hardware fill:#fce4ec,stroke:#c62828,stroke-width:1.5px,color:#7f1d1d
    classDef service  fill:#e8f5e9,stroke:#2e7d32,stroke-width:1.5px,color:#1b5e20
    classDef storage  fill:#fff9c4,stroke:#f9a825,stroke-width:1.5px,color:#856404
    classDef ai       fill:#f3e5f5,stroke:#6a1b9a,stroke-width:1.5px,color:#4a148c
    classDef cloud    fill:#e3f2fd,stroke:#1565c0,stroke-width:1.5px,stroke-dasharray:4 4,color:#0d47a1
    classDef internet fill:#fff3e0,stroke:#e65100,stroke-width:1.5px,stroke-dasharray:5 5,color:#bf360c
    classDef spacer   fill:none,stroke:none,color:transparent

    subgraph NUC ["🖥️ Intel NUC13 — Proxmox VE  |  i7-1360P · 64GB DDR4"]
        direction TB
        subgraph VM1 ["Ampitheatre  |  20GB · 4 cores"]
            S1[ ]:::spacer
            Plex[Plex Media Server]:::service
            Cal[Calibre-Web]:::service
            Med[Media Tooling internal only]:::service
        end
        subgraph VM2 ["Pygmachia  |  32GB · 6 cores"]
            Ptero[Pterodactyl Panel + Wings]:::service
            GS[Game Server Instances]:::service
            Ptero --> GS
        end
        subgraph VM3 ["The-Agora  |  8GB · 4 cores"]
            Mat[Matrix Synapse]:::service
        end
    end

    subgraph EXT ["🌐 External Access"]
        direction TB

        CFT([Cloudflare Zero Trust]):::cloud
        CFR([Cloudflare Realtime]):::cloud
        NGX([Nginx + SSL]):::cloud
        User([Remote User]):::internet
        CFT -->|email policy| User
        CFR --> User
        NGX --> User
    end

    subgraph PHYS ["🔌 Physical Network"]
        direction LR
        ISP[Virgin Media Hub 5 Modem Mode]:::hardware
        GW[Ubiquiti Gateway Ultra]:::hardware
        SW[Ubiquiti Switch Ultra 210W PoE]:::hardware
        ISP --> GW --> SW
    end

    subgraph ENDPTS ["💻 Endpoints"]
        direction TB
        PI[Pi 2B — Pi-hole PoE powered]:::hardware
        AP[Netgear R8000 Access Point]:::hardware
    end

    subgraph OBS ["🤖 Obsidian Tower — Ubuntu 24.04"]
        direction TB
        S2[ ]:::spacer
        S2b[ ]:::spacer
        S2c[ ]:::spacer
        OW[Open WebUI]:::ai
        OL[Ollama DeepSeek-R1 · Qwen2.5]:::ai
        SX[SearXNG]:::ai
        CUI[ComfyUI GTX 1070 Ti]:::ai
        OW --> OL
        OW --> SX
    end

    subgraph STORE ["💾 Storage"]
        direction TB
        NAS[("Synology DS920+ 2x16TB SHR")]:::storage
        BK[("Crucial X10 Pro 2TB SSD Backup")]:::storage
        NAS -.->|local backup| BK
    end

    SW --> NUC
    SW --> STORE
    SW -->|PoE| PI
    SW --> AP
    AP -->|Ethernet| OBS

    Plex -.->|NFS| NAS
    Cal  -.->|NFS| NAS

    Cal  -->|tunnel| CFT
    Mat  -->|tunnel| CFT
    Mat  <-.->|WebRTC| CFR
    Plex -->|Plex relay| User
    GS   -->|game traffic| NGX
