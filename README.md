# Home Lab: Self-Hosted Ubuntu Server

A spare Dell G3 laptop converted into a self-hosted home server running Ubuntu Server and Docker. It provides network-level ad blocking, private cloud storage, local speech-to-text and AI, media streaming, and a container dashboard, all on the home network.

> **Status:** Working personal project. Documented with the real problems I hit and how I fixed them.

## Table of Contents

- [Services](#services)
- [Hardware and Environment](#hardware-and-environment)
- [Architecture](#architecture)
- [Setup Overview](#setup-overview)
- [Example Configuration](#example-configuration)
- [Troubleshooting Highlights](#troubleshooting-highlights)
- [Security Notes](#security-notes)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)
- [What I Learned](#what-i-learned)

## Services

| Service | Purpose | Port | Deployment |
|---|---|---|---|
| [Pi-hole](https://pi-hole.net) | DNS-level ad and tracker blocking | 53, 80 | Docker (host network) |
| [Nextcloud](https://nextcloud.com) + MariaDB | Private cloud storage and phone photo backup | 8080 | Docker Compose |
| [Whisper ASR Webservice](https://github.com/ahmetoner/whisper-asr-webservice) | Local speech-to-text on the GPU | 9000 | Docker (NVIDIA runtime) |
| [Ollama](https://ollama.com) | Local language model (Mistral) | 11434 | Native install |
| [Open WebUI](https://github.com/open-webui/open-webui) | Browser chat interface for Ollama | 3000 | Docker |
| [Jellyfin](https://jellyfin.org) | Media streaming | 8096 | Docker |
| [Portainer](https://www.portainer.io) | Container management dashboard | 9443 | Docker |

## Hardware and Environment

- **Machine:** Dell G3 laptop
- **GPU:** NVIDIA GeForce GTX 1650 (4 GB VRAM)
- **RAM:** about 6.8 GB
- **Storage:** two drives of roughly 931 GB each. One holds the OS (LVM, 100 GB root volume); the other is formatted ext4 and mounted at `/mnt/storage` for data
- **OS:** Ubuntu Server 26.04
- **Network:** home network behind an ISP-supplied NBN modem; the server uses a fixed LAN address

## Architecture

```
Devices (laptop, phone)
        |  DNS queries
        v
  Pi-hole (port 53) ----> upstream DNS
  
Phone / browser
        |
        +--> Nextcloud (8080) --> MariaDB
        |         data stored on /mnt/storage
        +--> Jellyfin (8096)
        +--> Open WebUI (3000) --> Ollama (11434) --> GPU
        +--> Whisper ASR (9000) --> GPU
        +--> Portainer (9443) --> Docker socket
```

All services are reachable only inside the home network. Nothing is forwarded from the internet.

## Setup Overview

1. **Create installer media.** Download Ubuntu Server and write it to a USB drive with Balena Etcher (use the build for your OS).
2. **Install Ubuntu Server.** Guided storage with LVM, OpenSSH enabled.
3. **Prepare the data drive.**
   ```bash
   lsblk
   sudo parted /dev/sdb mklabel gpt
   sudo parted /dev/sdb mkpart primary ext4 0% 100%
   sudo mkfs.ext4 /dev/sdb1
   sudo mkdir -p /mnt/storage
   sudo mount /dev/sdb1 /mnt/storage
   sudo blkid | grep sdb1          # copy the UUID into /etc/fstab
   sudo systemctl daemon-reload
   sudo mount -a
   ```
4. **Install Docker** from Docker's official repository (see troubleshooting for repository notes on a very new Ubuntu release).
5. **Deploy services** with `docker run` or Docker Compose (examples below).
6. **Point devices at Pi-hole** by setting each device's DNS to the server's address (the ISP modem does not allow a custom DNS server).
7. **Install the NVIDIA Container Toolkit** for GPU workloads:
   ```bash
   sudo apt install nvidia-container-toolkit
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

## Example Configuration

Replace every `CHANGE_ME` value. **Never commit real passwords to a public repository.**

### Pi-hole

```bash
sudo docker run -d \
  --name pihole \
  --network host \
  -e TZ="Australia/Brisbane" \
  -e FTLCONF_webserver_api_password="CHANGE_ME" \
  -v ~/pihole/etc-pihole:/etc/pihole \
  -v ~/pihole/etc-dnsmasq.d:/etc/dnsmasq.d \
  --restart=unless-stopped \
  --cap-add=NET_ADMIN \
  pihole/pihole:latest
```

### Nextcloud and MariaDB (`docker-compose.yml`)

```yaml
services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    volumes:
      - ~/nextcloud/db:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: CHANGE_ME
      MYSQL_PASSWORD: CHANGE_ME
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud

  app:
    image: nextcloud:latest
    restart: unless-stopped
    ports:
      - 8080:80
    links:
      - db
    volumes:
      - /mnt/storage/nextcloud/data:/var/www/html
    environment:
      MYSQL_PASSWORD: CHANGE_ME
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_HOST: db
```

### Whisper ASR (GPU)

```bash
sudo docker run -d \
  --name whisper-webui \
  --gpus all \
  -p 9000:9000 \
  -v ~/whisper/audio:/app/audio \
  --restart unless-stopped \
  onerahmet/openai-whisper-asr-webservice:latest-gpu
```

API documentation is served at `http://<server-ip>:9000/docs`.

### Open WebUI (connects to Ollama)

```bash
sudo docker run -d \
  --name open-webui \
  -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v ~/open-webui:/app/backend/data \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
```

### Portainer

```bash
sudo docker volume create portainer_data
sudo docker run -d \
  --name portainer \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  --restart unless-stopped \
  portainer/portainer-ce:latest
```

## Troubleshooting Highlights

| Problem | Cause | Fix |
|---|---|---|
| Pi-hole running but ignoring DNS queries (`ignoring query from non-local network`) | Docker's default NAT changes the apparent source address | Run the container with `--network host` |
| `NO_PUBKEY` and `no installation candidate` for Docker packages | Key not imported correctly, typos in the repository file, and the repository had no packages for the newest Ubuntu codename | Download the key to a file, convert with `gpg --dearmor`, write the repository line in a text editor, use the previous supported codename |
| Whisper failed with `failed to discover GPU vendor from CDI` | NVIDIA Container Toolkit not installed | Install the toolkit and configure the Docker runtime |
| Whisper page unreachable | Application listens on port 9000, not the port published | Publish `-p 9000:9000` |
| Compose error: `additional properties 'environments' not allowed` | Typo in the key name | Use `environment:` |
| Second drive not found when formatting | Partition did not exist yet | Create a GPT label and partition with `parted`, then format |
| Nextcloud photos slow to load | First-time thumbnail generation plus slow mechanical disk | Let previews finish or run `occ preview:generate-all` |
| Locked out of the server | Forgotten password | Reset from GRUB recovery mode using `passwd` |
| Router would not accept a custom DNS server | ISP-locked modem | Set DNS per device |

Full write-up of each issue: see the project issues log.

## Security Notes

Done:
- Services are only exposed inside the home network.
- Containers isolate services from each other.
- Local AI keeps recordings and prompts on the home server.

Planned or recommended:
- Enable the host firewall (`ufw`) and allow only required ports.
- Move credentials out of Compose files into environment files or Docker secrets.
- Add a reverse proxy with proper TLS certificates.
- Keep an off-device backup copy and test restores.
- Restrict access to Portainer, since it has access to the Docker socket.

## Known Limitations

- Pi-hole cannot block YouTube ads (ads and video share the same domains).
- Router-wide DNS is not possible on the current ISP modem.
- Services currently use plain HTTP or self-signed certificates.
- Everything runs on one laptop, so heavy tasks (thumbnail generation, AI inference) can slow other services.

## Roadmap

- [ ] Reverse proxy with TLS
- [ ] Scheduled Nextcloud backups with a tested restore
- [ ] Enable and configure firewall rules
- [ ] Populate the Jellyfin media library
- [ ] Monitoring and update automation

## What I Learned

- Read the exact error message and the logs before changing anything.
- Paste long commands instead of typing them, and verify files with `cat` and `ls`.
- A working host driver does not guarantee GPU access inside containers.
- Container networking mode can change how services behave on the network.
- Documenting problems as they happen makes them easy to explain later.

## License

Documentation released under the [MIT License](LICENSE). Each service keeps its own license.
