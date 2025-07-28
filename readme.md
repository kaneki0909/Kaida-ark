# Kaida Ark Home Server Documentation

A detailed walkthrough of the architecture, setup, and configurations powering **Kaida Ark**, a GPU-enabled Proxmox VE home server built for AI workloads, media streaming, and advanced containerized services.



---

## 📌 1. System Overview
- **Host**: Acer Nitro 5 AN515-45-R4CL
- **OS**: Proxmox VE (Bare Metal Install)
- **GPU**: RTX 4070 Ti (PNY XLR8)
- **CPU**: Ryzen 7 7700X
- **RAM**: 32GB DDR5
- **Primary Network**: Wi-Fi (`wlp4s0`)
- **Use Cases**: AI inference (llama.cpp, Stable Diffusion), Plex Media Server, system monitoring

---

## 🌐 2. Network Configuration (vmbr1 + NAT)

### vmbr1 Configuration
Created a custom bridge `vmbr1` linked to Wi-Fi:
```bash
iface vmbr1 inet static
  address 10.10.x.1/24
  bridge_ports none
  bridge_stp off
  bridge_fd 0
  post-up iptables -t nat -A POSTROUTING -s 10.10.x.0/24 -o wlp4s0 -j MASQUERADE
  post-up echo 1 > /proc/sys/net/ipv4/ip_forward
  post-down iptables -t nat -D POSTROUTING -s 10.10.x.0/24 -o wlp4s0 -j MASQUERADE
```

### Enable IP Forwarding
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

---

## 📦 3. LXC Container Configuration

### Key Containers:
- `kaida-ent`: Internet-enabled container with static IP and DNS
- `kaida-gen`: For AI tools (Stable Diffusion, llama.cpp)

### DNS Fix:
Bind host resolv.conf:
```bash
mkdir -p /etc/systemd/resolved
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

### Static IP & Gateway in `/etc/network/interfaces` (inside container)
```bash
auto eth0
iface eth0 inet static
  address 10.10.x.10
  netmask 255.255.255.0
  gateway 10.10.x.1
```

---

## 🎮 4. GPU Passthrough

### Host Config:
- Enabled IOMMU in GRUB
- Created passthrough group for RTX 4070 Ti
- Verified with `lspci -nnv`

### LXC (if using GPU):
> Currently, GPU passthrough is active only on the host for AI workloads. Containers remain CPU-based unless passthrough is configured.

---

## 📊 5. Monitoring: Prometheus + Grafana

### Stack:
- **pve_exporter** running in a Python virtualenv
- **Prometheus** scraping metrics on port `9221`
- **Grafana** visualizing node/container status

### Services:
```bash
[Unit]
Description=pve_exporter

[Service]
User=root
ExecStart=/root/venv/bin/python /root/pve_exporter/exporter.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Metrics available at: `http://<proxmox-host-ip>:9221/metrics`

---

## 🎥 6. Plex Media Server (Inside Container)

- Installed in `kaida-ent`
- Mounted media storage via bind mount
- Configured `Preferences.xml` for secure token auth
- Verified access and playback on local network

---

## 🛠️ 7. Issues & Fixes

| Problem | Fix |
|--------|------|
| LXC has no internet | Static IP + proper NAT rules via `iptables` |
| DNS not resolving | Bound `/etc/resolv.conf` to systemd-resolved version |
| GPU not visible | Ensure IOMMU and passthrough group are correct |
| Prometheus can't scrape | Restart pve_exporter + verify port 9221 exposure |

---

## 📌 Notes
- All LXC containers are **unprivileged**
- Wi-Fi used as outbound interface via `wlp4s0`
- `vmbr1` enables NAT for full internet access
- Project name inspired by “Kaida,” the server identity 😄

---

> Last updated: July 2025
