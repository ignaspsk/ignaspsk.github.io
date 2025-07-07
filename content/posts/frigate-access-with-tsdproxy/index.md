---
title: "How to set up Frigate remote access via Tailscale using TSDProxy"
date: 2025-03-01
draft: false
description: "example description"
tags: ["frigate", "tailscale", "tsdproxy"]
showTableOfContents: true
showComments: true
---
## Introduction
In case you wanted to have remote access to your docker services, there are [tailscale sidecars](https://tailscale.com/blog/docker-tailscale-guide) for that, however it has a drawback - you need a new sidecar for each new service. If only there was a Tailscale Docker Proxy that would manage multiple docker services without the need of dedicated sidecars...

Well, there is. And it's called [TSDProxy](https://almeidapaulopt.github.io/tsdproxy/docs/getting-started/). Tailscale even uploaded a [video](https://youtu.be/5lJrXEXF8eM) dedicated to this software. Another perk is that you can have tailscale installed on your host machine (e.g. for SSH) while having TSDProxy running without additional configuration, contrary to tailscale sidecars, which didn't work for me.

One of the services you might want to have in you tailnet is [Frigate](https://docs.frigate.video/), but setting it up was a bit tricky for me. I wish I could have found a complete guide on how to do it, as it would have saved me a lot of time. So I made this guide, and hopefully someone will find it useful.

In this guide I will cover how to:
- How to install TSDProxy
- How to install Frigate and connect it to TSDProxy (You will need to refer to Frigate documentation to write your config.yaml file, I will not cover it)
## Prerequisites

- Ubuntu or Debian server (I tested on [Ubuntu 24.04.2 LTS Server](https://ubuntu.com/download/server)), or Rasbian on RPi5 (tested)
- Docker installed ([Downloads page](https://docs.docker.com/engine/install/))
- SSH or otherwise direct access to machine you're planning to run Frigate on.
- Recommended to enable HTTPS Certificates in [DNS settings](https://login.tailscale.com/admin/dns) of tailnet.

## Installation

### Installing TSDProxy
Referring to the [documentation](https://almeidapaulopt.github.io/tsdproxy/docs/getting-started/), follow these steps.

1. Create docker-compose.yaml for TSDProxy and insert yaml.
```bash
echo "services:
  tsdproxy:
    image: almeidapaulopt/tsdproxy:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - datadir:/data
      - <PATH_TO_YOUR_CONFIG_DIR>:/config
    restart: unless-stopped
    ports:
      - "8080:8080"

volumes:
  datadir:" >> docker-compose.yaml
```

2. Change `<PATH_TO_YOUR_CONFIG_DIR>` to directory that you want, e.g. `/home/username/tsdproxy` using your favorite text editor.
```bash
nano docker-compose.yaml
```

3. After that, start TSDProxy docker container with `-d` flag to run it in the background.
```bash
sudo docker compose up -d
```

4. A configuration file `~/directory-of-your-choice/tsdproxy.yaml` is created and populated.
```tsdproxy.yaml
defaultProxyProvider: default
docker:
  local: # name of the docker target provider
    host: unix:///var/run/docker.sock # host of the docker socket or daemon
    targetHostname: 172.31.0.1 # hostname or IP of docker server
    defaultProxyProvider: default # name of which proxy provider to use
files: {}
tailscale:
  providers:
    default: # name of the provider
      authKey: "" # optional, define authkey here
      authKeyFile: "" # optional, use this to load authkey from file. If this is defined, Authkey is ignored
      controlUrl: https://controlplane.tailscale.com # use this to override the default control URL
  dataDir: /data/
http:
  hostname: 0.0.0.0
  port: 8080
log:
  level: info # set logging level info, error or trace
  json: false # set to true to enable json logging
proxyAccessLog: true # set to true to enable container access log
```

5. Make sure that `targetHostname` is same as in the output of the following command.
```bash
sudo docker network inspect bridge | grep Gateway
```

6. Insert your authkey. Get it [here](https://login.tailscale.com/admin/settings/keys). Make sure you enable "reusable" when generating the authkey.
7. When you're done editing, restart to apply changes.
```bash
sudo docker compose restart
```
8. Optional: run a sample service to make sure everything is working
```bash
docker run -d --name sample-nginx -p 8111:80 --label "tsdproxy.enable=true" nginx:latest
```

If `sample-nginx` appears in your [dashboard](https://login.tailscale.com/admin/machines), you're good to go.

### Installing Frigate
In case of fire, refer to the [Frigate documentation](https://docs.frigate.video/guides/getting_started).


1. Create another directory. E.g. `~/frigate`
```bash
mkdir ~/frigate && cd ~/frigate
```

2. You will need to create two folders and `docker-compose.yml` in the same directory.
```bash
mkdir storage config && touch docker-compose.yml
```

3. Open `docker-compose.yml` and paste the following. (`CTRL+SHIFT+V` to paste in the terminal)
```docker-compose.yml
services:
  frigate:
    container_name: frigate
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    volumes:
      - ./config:/config
      - ./storage:/media/frigate
      - type: tmpfs # Optional: 1GB of memory, reduces SSD/SD Card wear
        target: /tmp/cache
        tmpfs:
          size: 1000000000
#    devices:
#      - /dev/apex_0:/dev/apex_0 # If you have coral tpu
    ports:
      - "8971:5000"
      - "8554:8554" # RTSP feeds
    labels: # Important otherwise will not connect to tsdproxy
      tsdproxy.enable: "true"
      tsdproxy.scheme: "http"
      tsdproxy.tlsvalidate: "false"
```

4. Start the container.
```bash
sudo docker compose up -d
```

5. View logs to find admin password for Frigate. Admin username is always `admin`
```bash
sudo docker logs frigate
```

Now you should see "frigate" machine in your [dashboard](https://login.tailscale.com/admin/machines). Approve it, access it in your tailnet's FQDN. It can take a while before TSDProxy connects, generates certificates, starts the proxy.

Once you're in, click the settings button and head over to Configuration Editor to finish configuring your Frigate instance. [Here's where you should continue](https://docs.frigate.video/guides/getting_started/#configuring-frigate).

Cover background by [rrricharddd](https://unsplash.com/@rrricharddd) on [Unsplash](https://unsplash.com/photos/a-room-with-a-bunch-of-wires-on-the-wall-PNbfQ3Wyiws)
