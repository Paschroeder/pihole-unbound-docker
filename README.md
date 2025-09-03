# Why another fork
This is just an version with the newest pihole and unbound versions with all the adaption of the env vars which was changing after pihole 2024.07.x

# Homelab DNS Service Setup

This guide explains how to set up Pi-hole and Unbound DNS services to run automatically as a system service on your homelab server.

## Overview

We created a systemd service that automatically starts your Docker Compose DNS stack (Pi-hole + Unbound) at boot time, with proper user separation and deployment workflow.

## Directory Structure

```
/home/<user>/repos/homelab-config/dns/    # Git repository (development)
├── docker-compose.yml
├── .env

/home/homelab/dns/                    # Production deployment
├── docker-compose.yml                # Copied from repo
├── .env                              # Copied from repo  
└── data/                             # Copied from repo
    ├── pihole/
    └── dnsmasq.d/
```

## User Setup

1. **Create dedicated service user:**
   ```bash
   sudo useradd -r -s /bin/false -d /home/homelab homelab
   sudo usermod -aG docker homelab
   sudo mkdir -p /home/homelab/dns
   sudo chown -R homelab:homelab /home/homelab
   ```

2. **Create data directories with proper ownership:**
   ```bash
   # Create the data structure
   sudo mkdir -p /home/homelab/dns/data/{pihole,dnsmasq.d}
   
   # Set proper ownership (important for container permissions)
   sudo chown -R homelab:homelab /home/homelab/dns/
   
   # Set appropriate permissions
   sudo chmod -R 755 /home/homelab/dns/data/
   ```

3. **Allow <user> to deploy:**
   ```bash
   sudo usermod -a -G homelab <user>
   ```

## Systemd Service

**Created:** `/etc/systemd/system/homelab-dns.service`

```ini
[Unit]
Description=Homelab DNS Services (Pi-hole + Unbound)
Requires=snap.docker.dockerd.service
After=snap.docker.dockerd.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
User=homelab
Group=docker
WorkingDirectory=/home/homelab/dns
ExecStartPre=/bin/bash -c 'until /snap/bin/docker info >/dev/null 2>&1; do echo "Waiting for Docker..."; sleep 2; done'
ExecStart=/snap/bin/docker-compose up -d
ExecStop=/snap/bin/docker-compose down
TimeoutStartSec=300
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Enable service:**
```bash
sudo systemctl enable homelab-dns.service
sudo systemctl daemon-reload
```

## Deployment Script

**Created:** `/home/<user>/scripts/deploy-dns.sh`

```bash
#!/bin/bash
# Copy all files including .env (hidden files)
sudo cp -r /home/<user>/repos/homelab-config/dns/. /home/homelab/dns/

# Set proper ownership
sudo chown -R homelab:homelab /home/homelab/dns/

# Restart service
sudo systemctl restart homelab-dns.service
```

**Make executable:**
```bash
chmod +x /home/<user>/scripts/deploy-dns.sh
```

## Docker Compose Configuration

**Key points:**
- Use relative paths for volumes: `./data/pihole/pihole:/etc/pihole/`
- Environment variables in `.env` file
- Restart policy: `restart: unless-stopped`

## How It Works

1. **Development:** Edit files in `/home/<user>/repos/homelab-config/dns/`
2. **Deploy:** Run `./deploy-dns.sh` to copy to production
3. **Auto-start:** Service starts automatically at boot
4. **Management:** Use standard systemctl commands

## Common Commands

```bash
# Deploy changes
./deploy-dns.sh

# Check service status
sudo systemctl status homelab-dns.service

# View logs
journalctl -fu homelab-dns.service

# Manual start/stop/restart
sudo systemctl start homelab-dns.service
sudo systemctl stop homelab-dns.service
sudo systemctl restart homelab-dns.service

# Check running containers
docker ps
```

# Pi-hole & Unbound DNS Docker Setup

This is a docker compose setup which starts a [Pi-hole](https://pi-hole.net/) and [nlnetlab's Unbound](https://nlnetlabs.nl/projects/unbound/about/) as upstream recursive DNS using official (or ready-to-use) images. The main idea here is to add security, [privacy](https://www.cloudflare.com/learning/dns/what-is-recursive-dns/) and have ad and malware protection, everything hosted locally.

If you want to learn more about why you want to have exactly this setup, read a [detailed explanation here](https://docs.pi-hole.net/guides/dns/unbound/).

## Prepare

### Understand the Setup

This setup works on a machine that does not itself already has DNS running (i.e. port 53 is already used). If you have a setup like that (e.g. running on a Synology NAS with a Directory Server), you would need a setup that creates a [Mac VLAN](https://docs.docker.com/network/macvlan/) so the container appears with a different IP. In this [case check out this example here](https://github.com/chriscrowe/docker-pihole-unbound/tree/main/two-container).

It is designed to have 2 containers running next to each other and do not aim to combine both programs in one. The idea is to minimize the work needed to adapt provided containerized versions of Pi-hole and Unbound, i.e. use the official images, therefore making it easier to upgrade each.

### Prerequisites

First you need a recent version of [Docker installed](https://docs.docker.com/get-docker/) which at least supports Docker compose v2.
Further you may want to have a [server or IoT device](https://docs.pi-hole.net/main/prerequisites/) where this stack can run on, since this should be reachable by every other client 24/7.
Finally, don't forget to change your [default DNS server to the server IPs address of your server](https://docs.pi-hole.net/main/post-install/).

### Configuration

The main configuration can be set in the `.env` file which overwrites the ENV variables in the `docker-compose.yml` - change it to your liking:

```properties
WEBPASSWORD= # set the password to use in the Web Admin UI
HOST_IP_V4_ADDRESS= # the IP of the host the Pi-hole runs on - defaults to localhost
TIMEZONE= # set your timezone (used to schedule cron jobs e.g.)
```

## Deploy Containers

[![asciicast](https://asciinema.org/a/581383.svg)](https://asciinema.org/a/581383)

Start the stack with going to the root of the repo and do:

```bash
 docker compose up --build -d --remove-orphans
```

To stop everything:

```bash
 docker compose down
```

Pro-Tip, if you want to directly deploy to a remote you can do

```bash
 docker compose -H "ssh://your-remote-host" up --build -d --remove-orphans
```

## Use Web UI

If you didn't change anything and start this on your local machine you can access the Pi-hole web ui with

```
http://localhost:8080/admin/
```

The default password is `changeMeNow`.

## Test Setup

To test if Pi-Hole with unbound is working correctly you can use the test domain `unboundpiholetestdomain.org` I set up in Unbound.
In your terminal (you might [need to install `nslookup`](https://www.tecmint.com/install-dig-and-nslookup-in-linux/)) do:

```
nslookup unboundpiholetestdomain.org localhost
```
This command will use localhost as DNS, if you are running it on a different machine, use the appropriate IP.

This should return the IP `192.168.123.123`:
```
Server:         localhost
Address:        ::1#53

Name:   unboundpiholetestdomain.org
Address: 192.168.123.123
```

if setup correctly it should also work without forcing DNS

```
nslookup unboundpiholetestdomain.org
```

## Advanced

### Persistence after Restart

By default, Pi-hole will forget everything after a restart of the docker container. To change that you need to set
a docker volume to show Pi-hole where to save the configuration. You need to map `/etc/Pi-hole/` and `/etc/dnsmasq.d/` to
a directory on the server. [Read here if you want to learn more about volumes](https://stackoverflow.com/questions/68647242/define-volumes-in-docker-compose-yaml).

There is an example in the `docker-compose.yml`:

```yaml
services:
  Pi-hole:
    container_name: ...
# RECOMMENDED: Uncomment and adapt if you want to persist Pi-hole configurations after restart
#    volumes:
#      - "/var/lib/docker/volumes/pihole/pihole:/etc/pihole/"
#      - "/var/lib/docker/volumes/pihole/dnsmasq.d:/etc/dnsmasq.d/"
```

### Pi-hole Configurations

In the `docker-compose.yml` you can add or change the Pi-hole Docker standard configuration variables in

```yaml
services:
  Pi-hole:
    container_name: ...
    environment:
      # here
```
Check out [possible configurations here](https://github.com/pi-hole/docker-pi-hole).

Additionally, you can change various settings in your Pi-hole instance (e.g. the used ad-list) through the web ui. I won't
get into detail here apart from recommending `https://v.firebog.net/hosts/lists.php` as a good default starting list.


### Upgrade Base Images

In the `docker-compose.yml` change the used Pi-hole version by changing

```yaml
services:
  Pi-hole:
    container_name: ...
    image: Pi-hole/Pi-hole:2023.03.1 # <- update image version here, see: https://github.com/pi-hole/docker-pi-hole/releases
    hostname: ...
```

and Unbound by changing the `FROM` in `./unbound/Dockerfile` 

```dockerfile
# Update the version here, I use the docker build from https://github.com/MatthewVance/unbound-docker
FROM mvance/unbound:1.17.1
...
```

### Define Local A-Records 

If you want to resolve certain domains locally you can set A-Records in `./unbound/conf/a-records.conf`. There are already examples, but to add a new record do:

```
# Example: Resolve all *.mysite.com addresses to the same ip of the main reverse proxy / router

local-zone: "mysite.com." redirect
local-data: "mysite.com. 86400 IN A 192.168.1.1"
```

Check here the [full documentation](https://unbound.docs.nlnetlabs.nl/_/downloads/en/latest/pdf/) or [tutorial](https://calomel.org/unbound_dns.html) to learn more.

### Unbound, Forwarders and Manual Configuration

Unbound is set as a recursive DNS, because all forwarders in `./unbound/conf/a-records.conf` are commented out. If you prefer to use cloudflare or any other public DNS as upstream instead of having the slight performance impact of directly asking the nameservers, then you can enable the respective server by removing the comment (but then using Unbound at all has little value.

If you want to fine-tune the Unbound configuration, you can add the file `./unbound/conf/unbound.conf` (see an [example here](https://github.com/MatthewVance/unbound-docker/blob/master/unbound.conf)) and Unbound will use it.

## Limitations

### Supported Platforms

Currently, this setup will only support platform type `amd64`, that means it will not run on machines that e.g. have an [ARM architecture](https://en.wikipedia.org/wiki/ARM_architecture_family) like the [Raspberry Pi](https://www.raspberrypi.com/documentation/computers/processors.html). While the official Pi-hole image supports [multi-arch](https://www.docker.com/blog/multi-arch-build-and-images-the-simple-way/), MatthewVance's unbound image does not. There is, however, a solution: there is a specific build for `arm/v7` which can be found on [Docker hub](https://hub.docker.com/r/mvance/unbound-rpi/tags). Just update the `Dockerfile` in `./unbound/Dockerfile`:

```dockerfile
FROM mvance/unbound-rpi:1.17.1
...
```

### Watchtower

If you use tools like [Watchtower](https://github.com/containrrr/watchtower) to be notified about image updates - this will not work with Unbound here since we re-build it to create a self-contained, stateless image. It is possible to use the image `mvance/unbound` directly in the `docker-compose` and mount the configuration files to unbound instead of pre-building it. See [MatthewVance readme](https://github.com/MatthewVance/unbound-docker) on how to do that.

# Links

## Credits

* [nlnetlabs Unbound](https://nlnetlabs.nl/projects/unbound/about/) (BSD license)
* [MatthewVance's Unbound Docker Image](https://github.com/MatthewVance/unbound-docker) (MIT License)
* [Pi-hole](https://github.com/pi-hole/pi-hole) (European Union Public License)
* [Official Pi-hole Docker Image](https://github.com/pi-hole/docker-pi-hole) (unknown license)

## Similar Projects

* [chriscrowe's Mac Vlan Setup](https://github.com/chriscrowe/docker-pihole-unbound)
* [origamiofficial's One Container Solution](https://github.com/origamiofficial/docker-pihole-unbound)
* [JD10NN3's Solution](https://github.com/JD10NN3/docker-pihole-unbound)

## Further Information

* [Pi-hole Documentation](https://docs.pi-hole.net/)
* [Unbound Documentation](https://unbound.docs.nlnetlabs.nl/_/downloads/en/latest/pdf/)
* [Pi-hole + Unbound Details](https://docs.pi-hole.net/guides/dns/unbound/)
* [How to run docker-compose on remote host?](https://stackoverflow.com/questions/35433147/how-to-run-docker-compose-on-remote-host) 

# License

Copyright 2023 Patrick Favre-Bulle

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

```
https://www.apache.org/licenses/LICENSE-2.0
```

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
