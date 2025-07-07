---
title: "How to install PCIe Coral TPU drivers in Ubuntu 24.04.2 LTS Server"
date: 2025-03-01
draft: false
description: "example description"
tags: ["coral", "ubuntu"]
showTableOfContents: true
showComments: true
---
## Introduction
If you followed the [official coral documentation](https://coral.ai/docs/m2/get-started), you will get error when installing the `gasket-dkms` package. That's because it's actually outdated, as well as the instructions of adding their repo to apt.
Hence I wrote a this guide, I hope you will find it helpful!

## Prerequisites

- Installed [Ubuntu 24.04.2 server](https://ubuntu.com/download/server). Debian server should be fine, but the 4th step of Installation section may need to be adjusted to your OS.
- Installed PCIe Coral Edge TPU in a PCIe slot (A+E in my case)
- If you have secure boot on, during installation you will get a dialog that will guide you on how to enroll new MOK keys to enable third-party drivers. You will need to reboot.
- This is done on an amd64 computer, but not tested on a Raspberry Pi, YMMV.
## Installation

### Compiling gasket from source
The version linked in the official coral documentation doesn't work on recent kernel versions. The updated version is available at [gasket-driver repo](https://github.com/google/gasket-driver/), but has no pre-built packages there. 
This is where [gasket-builder](https://github.com/jnicolson/gasket-builder) comes in:
> This repository contains a Docker file to build the deb package to install on recent kernel versions, to avoid installing development dependencies on production servers.

1. Make sure you have docker installed as per [Docker documentation](https://docs.docker.com/engine/install/ubuntu/)
2. Clone git repo
```bash
git clone https://github.com/jnicolson/gasket-builder
```
3. Move to gasket builder directory and start building. Use sudo if your user isn't in the docker group.
```bash
cd gasket-builder
docker build --output . .
```
4. If building succeeds, you should see `gasket-dkms<version>.deb`, where `<version>` is version of the driver.
```bash
ls
```
### Installing .deb file from terminal
This is self-explanatory.
```bash
sudo apt install ./gasket-dkms<version>.deb
```
If you get "unsandboxed" error, you should still be fine to proceed. This is just a minor apt sandbox artifact (common when installing from home directories) but has no impact on functionality.

### Adding Googles repository
Commands provided in the official docs give errors because it is outdated. Use these instead.

1. Add repository with signed-by clause
```bash
echo "deb [signed-by=/etc/apt/keyrings/coral.gpg] https://packages.cloud.google.com/apt coral-edgetpu-stable main" | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
```

2. Create keyrings directory if needed
```bash
sudo mkdir -p /etc/apt/keyrings
```

3. Download key
```bash
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/coral.gpg
```

4. Override Ubuntu's package priority. This may be needed because Ubuntu's repositories have older versions of the same packages. This creates a preferences file to prioritize Google's repo.
```bash
echo "Package: *
Pin: origin packages.cloud.google.com
Pin-Priority: 1001" | sudo tee /etc/apt/preferences.d/coral-priority
```

5. Update apt
```bash
sudo apt update
```

### Installing the libedgetpu1-std
Installing from Google's repository should work just fine. Just to make sure it comes from the right repository, we will use `-t coral-edgetpu-stable` flag.
```bash
sudo apt install -t coral-edgetpu-stable libedgetpu1-std
```

### Verify functionality
If you don't want to reboot your system, load modules manually with `modprobe`.
```bash
sudo modprobe gasket
sudo modprobe apex
```

Check if loaded.
```bash
lsmod | grep -E 'gasket|apex'
```

And final check.
```bash
ls /dev/apex_0
```
If you see `/dev/apex_0` in the output, then you have successfully installed the driver and Edge TPU runtime.

Cover photo made by me.
